<!-- $theme: default -->

Switching to the new EngineFacade
=================================

by Roman Podoliaka

@rpodoliaka
irc: **rpodolyaka**
rpodolyaka@mirantis.com


slides: http://podoliaka.org/talks/

---

The new EngineFacade
====================

* spec: https://specs.openstack.org/openstack/oslo-specs/specs/kilo/make-enginefacade-a-facade.html
* implemented by **zzzeek** (Mike Bayer) in Liberty (oslo.db >= 1.12.0)
* a new clean API for using of oslo.db
	* thread-safe initialization
	* declarative management of session / connection scope
	* easier offloading of read-only transactions to asynchronous replicas

---

### "old" EngineFacade: thread-safe initialization is left up to users

```python
_ENGINE_FACADE = None
_LOCK = threading.Lock()

def _create_facade_lazily():
    global _LOCK, _ENGINE_FACADE
    if _ENGINE_FACADE is None:
        with _LOCK:
            if _ENGINE_FACADE is None:
                _ENGINE_FACADE = db_session.EngineFacade.from_config(CONF)
    return _ENGINE_FACADE
```

---

### "new" EngineFacade: lazy thread-safe initialization

```python
from oslo.db import enginefacade as sql

@sql.reader
def some_api_method(context):
    # work with context.session

@sql.writer
def some_other_api_method(context):
    # work with context.session
```

---

### "new" EngineFacade: multiple instances of EngineFacade

```python
main_context_manager = sql.transaction_context()
api_context_manager = sql.transaction_context()
```
---

### "new" EngineFacade: project specific configuration

```python
context_manager = enginefacade.transaction_context()

context_manager.configure(sqlite_fk=True)
context_manager.append_on_engine_create(set_hook)
```

---

### "old" EngineFacade: get_session() boilerplate

```python
@require_context
@handle_db_data_error
def snapshot_create(context, values):
    values['snapshot_metadata'] = \
    	_metadata_refs(values.get('metadata'),
         models.SnapshotMetadata)
    if not values.get('id'):
        values['id'] = str(uuid.uuid4())

    session = get_session()
    with session.begin():
        snapshot_ref = models.Snapshot()
        snapshot_ref.update(values)
        session.add(snapshot_ref)

        return _snapshot_get(context, values['id'],
                             session=session)
```

---

### "old" EngineFacade: private DB API functions

```python
@require_context
def _snapshot_metadata_get(context, snapshot_id,
                           session=None):
    rows = _snapshot_metadata_get_query(
    	context,snapshot_id, session).all()
    result = {}
    for row in rows:
        result[row['key']] = row['value']

    return result
```

---

### "new" EngineFacade: session scope is managed declaratively

```python
@main_context_manager.writer
def quota_destroy_all_by_project_and_user(context,
                                          project_id,
                                          user_id):
    model_query(context, models.ProjectUserQuota,
                read_deleted="no").\
        filter_by(project_id=project_id).\
        filter_by(user_id=user_id).\
        soft_delete(synchronize_session=False)
```

---

What projects are already using the new EngineFacade?
=====================================================

* [Nova](https://review.openstack.org/#/q/status:merged+project:openstack/nova+branch:master+topic:bp/new-oslodb-enginefacade)
* [Neutron](https://review.openstack.org/#/q/status:merged+project:openstack/neutron+branch:master+topic:bp/enginefacade-switch)
* [Ironic](https://review.openstack.org/#/q/status:merged+project:openstack/ironic+branch:master+topic:enginefacade)
* [Keystone](https://review.openstack.org/#/c/257458/)
* ...

---

Need help?
==========

Slides:  http://podoliaka.org/talks/

Bugs: https://bugs.launchpad.net/oslo.db

Questions:

* IRC
	* #openstack-oslo at freenode
	* **zzzeek** (Mike Bayer)
	* **rpodolyaka** (Roman Podoliaka)
* ML
	* http://lists.openstack.org/cgi-bin/mailman/listinfo/openstack-dev
	* tag with **[oslo][db]**
