+++
draft = false
date = 2025-05-19
title = "debuginfod: easier debugging on Linux"
tags = ["debugging"]
+++

# What happened

Since 2022, popular Linux distros ([Arch Linux](https://debuginfod.archlinux.org/),
[Fedora](https://debuginfod.fedoraproject.org/), [Ubuntu](https://debuginfod.ubuntu.com),
[Debian](https://debuginfod.debian.net)) have been serving DWARF[^dwarf] debug
symbols and the source code for the packages in their repositories using
[debuginfod](https://sourceware.org/elfutils/Debuginfod.html).

debuginfod is a web service that distributes ELF binaries, DWARF debug symbols, and source code
over HTTP. Debuggers, profilers, and other tools can now _lazily_ fetch that information from a
remote server to help you analyze program crashes, reliably display stack traces, and so on.

Previously, you would need to manually install additional packages that contained "stripped"
debug symbols, and some Linux distros (most notably, Arch Linux) did not even provide those,
which forced people to build everything from source.

[^dwarf]: The data format for debug symbols used in ELF binaries is aptly named DWARF.

# Why do we need debug symbols

Debug symbols are like bread crumbs left by the compiler, so that debug tools can introspect
the state of a running program (or a coredump). They are essential for figuring out the location
of variables in memory (or CPU registers), mapping generated instructions back to the source code,
and even computing stack traces (more on that in a separate blog post!).

Without debug symbols, you would only be able to tell which assembler instruction is currently being
executed, but not necessarily the function name, or the specific line of code it corresponds to.
There would be no way to print the value of a variable by name — you would have to manually figure
out its location in memory by analyzing the assembler listing for the function in question.

Consider the following example of debugging the `python` binary without debug symbols:

```text
~> gdb /usr/bin/python
(No debugging symbols found in /usr/bin/python)

(gdb) run -c "import time; time.sleep(60)"
Starting program: /usr/bin/python -c "import time; time.sleep(60)"
^C
Program received signal SIGINT, Interrupt.
0x00007ffff76a2006 in ?? () from /usr/lib/libc.so.6

(gdb) bt
#0  0x00007ffff76a2006 in ?? () from /usr/lib/libc.so.6
#1  0x00007ffff76f2a92 in clock_nanosleep () from /usr/lib/libc.so.6
#2  0x00007ffff7ad2129 in ?? () from /usr/lib/libpython3.13.so.1.0
#3  0x00007ffff798ff94 in ?? () from /usr/lib/libpython3.13.so.1.0
#4  0x00007ffff795f82d in PyObject_Vectorcall () from /usr/lib/libpython3.13.so.1.0
#5  0x00007ffff796ecd4 in _PyEval_EvalFrameDefault () from /usr/lib/libpython3.13.so.1.0
#6  0x00007ffff7a41695 in PyEval_EvalCode () from /usr/lib/libpython3.13.so.1.0
#7  0x00007ffff7a7f433 in ?? () from /usr/lib/libpython3.13.so.1.0
#8  0x00007ffff7a7c81a in ?? () from /usr/lib/libpython3.13.so.1.0
#9  0x00007ffff7a77f3e in ?? () from /usr/lib/libpython3.13.so.1.0
#10 0x00007ffff7a77da3 in ?? () from /usr/lib/libpython3.13.so.1.0
#11 0x00007ffff7a7733b in Py_RunMain () from /usr/lib/libpython3.13.so.1.0
#12 0x00007ffff7a2e95c in Py_BytesMain () from /usr/lib/libpython3.13.so.1.0
#13 0x00007ffff76376b5 in ?? () from /usr/lib/libc.so.6
#14 0x00007ffff7637769 in __libc_start_main () from /usr/lib/libc.so.6
#15 0x0000555555555045 in _start ()
```

Note that:

* Function names are missing
* Arguments of functions, their data types and values are unknown (the same is true for local variables not shown here)
* Source code is unavailable

And now compare it to the snapshot of a debugging session with DWARF symbols:

```text
~> gdb /usr/bin/python
Reading symbols from /home/malor/.cache/debuginfod_client/67ed9ffc907ba51c98b0a07244e6cba2bb62ec04/debuginfo...

(gdb) run -c "import time; time.sleep(60)"
Starting program: /usr/bin/python -c "import time; time.sleep(60)"
^C
Program received signal SIGINT, Interrupt.
__internal_syscall_cancel (a1=a1@entry=1, a2=a2@entry=1, a3=a3@entry=140737488346272, a4=a4@entry=0, a5=a5@entry=0, a6=a6@entry=0, nr=230) at cancellation.c:44
44            return result;

(gdb) bt
#0  __internal_syscall_cancel (a1=a1@entry=1, a2=a2@entry=1, a3=a3@entry=140737488346272, a4=a4@entry=0, a5=a5@entry=0, a6=a6@entry=0, nr=230) at cancellation.c:44
#1  0x00007ffff76f2a92 in __GI___clock_nanosleep (clock_id=<optimized out>, clock_id@entry=1, flags=flags@entry=1, req=req@entry=0x7fffffffdca0, rem=rem@entry=0x0) at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:48
#2  0x00007ffff7ad2129 in pysleep (timeout=<optimized out>) at ./Modules/timemodule.c:2262
#3  time_sleep (self=<optimized out>, timeout_obj=<optimized out>) at ./Modules/timemodule.c:408
#4  0x00007ffff798ff94 in cfunction_vectorcall_O (func=0x7ffff6e054e0, args=0x7ffff7f92078, nargsf=<optimized out>, kwnames=0x0) at ./Include/cpython/methodobject.h:50
#5  0x00007ffff795f82d in _PyObject_VectorcallTstate (tstate=0x7ffff7d25e90 <_PyRuntime+283024>, callable=0x7ffff6e054e0, args=0x7ffff7f92078, nargsf=9223372036854775809, kwnames=0x0) at ./Include/internal/pycore_call.h:168
#6  PyObject_Vectorcall (callable=0x7ffff6e054e0, args=0x7ffff7f92078, nargsf=9223372036854775809, kwnames=0x0) at Objects/call.c:327
#7  0x00007ffff796ecd4 in _PyEval_EvalFrameDefault (tstate=<optimized out>, frame=<optimized out>, throwflag=<optimized out>) at Python/generated_cases.c.h:813
#8  0x00007ffff7a41695 in PyEval_EvalCode (co=0x7ffff6e30c30, globals=<optimized out>, locals=0x7ffff6e34580) at Python/ceval.c:604
#9  0x00007ffff7a7f433 in run_eval_code_obj (tstate=tstate@entry=0x7ffff7d25e90 <_PyRuntime+283024>, co=co@entry=0x7ffff6e30c30, globals=globals@entry=0x7ffff6e34580, locals=locals@entry=0x7ffff6e34580) at Python/pythonrun.c:1381
#10 0x00007ffff7a7c81a in run_mod (mod=mod@entry=0x5555556066b0, filename=filename@entry=0x7ffff6e346f0, globals=globals@entry=0x7ffff6e34580, locals=locals@entry=0x7ffff6e34580, flags=flags@entry=0x7fffffffe158, arena=arena@entry=0x7ffff6f1bd10,
    interactive_src=0x7ffff6e1e830, generate_new_source=0) at Python/pythonrun.c:1466
#11 0x00007ffff7a77f3e in _PyRun_StringFlagsWithName (str=str@entry=0x7ffff6e34690 "import time; time.sleep(60)\n", name=name@entry=0x7ffff6e346f0, start=start@entry=257, globals=globals@entry=0x7ffff6e34580, locals=locals@entry=0x7ffff6e34580,
    flags=flags@entry=0x7fffffffe158, generate_new_source=0) at Python/pythonrun.c:1261
#12 0x00007ffff7a77da3 in _PyRun_SimpleStringFlagsWithName (command=0x7ffff6e34690 "import time; time.sleep(60)\n", name=name@entry=0x7ffff7afd6a6 "<string>", flags=flags@entry=0x7fffffffe158) at Python/pythonrun.c:574
#13 0x00007ffff7a7733b in pymain_run_command (command=<optimized out>) at Modules/main.c:253
#14 pymain_run_python (exitcode=0x7fffffffe14c) at Modules/main.c:687
#15 Py_RunMain () at Modules/main.c:775
#16 0x00007ffff7a2e95c in Py_BytesMain (argc=<optimized out>, argv=<optimized out>) at Modules/main.c:829
#17 0x00007ffff76376b5 in __libc_start_call_main (main=main@entry=0x555555555120 <main>, argc=argc@entry=3, argv=argv@entry=0x7fffffffe3b8) at ../sysdeps/nptl/libc_start_call_main.h:58
#18 0x00007ffff7637769 in __libc_start_main_impl (main=0x555555555120 <main>, argc=3, argv=0x7fffffffe3b8, init=<optimized out>, fini=<optimized out>, rtld_fini=<optimized out>, stack_end=0x7fffffffe3a8) at ../csu/libc-start.c:360
#19 0x0000555555555045 in _start ()
```

All the binaries (executables and libraries) are exactly the same. The only difference is whether
or not debug symbols are loaded. When they _are_ loaded, gdb is able to provide rich information
about all frames in the call stack, including the function argument values and the lines of code
being executed.

# So what is the problem

Debug information is generated at compile time and is _specific_ to that particular build
(because optimization levels and other compiler options significantly affect code generation,
inlining of function calls, location of variables in memory, etc).

Contrary to popular belief, you don't need a separate "debug" build[^debug] to troubleshoot a
program (otherwise, how would you troubleshoot "production" builds?); a compiler will happily
generate debug symbols for an optimized binary. Similarly, having additional debug information
in the binary does _not_ incur performance overhead.

*If that is true, why not have debug symbols embedded in binaries at all times to simplify debugging?*

The problem with debug symbols is their size — it's not uncommon to see overhead on the order
of hundreds of percent. The usual solution is to _strip_ debug symbols and ship them separately
[^separately]. This way, the "normal" packages remain lean, and people who need debug symbols
can fetch and install them independently. But you have to do it manually and make sure you get
the correct version.

debuginfod gives you the best of both worlds:

* Binaries are packaged and shipped without debug symbols, and thus, remain small
* Debug symbols are automatically fetched on demand

[^debug]: To be clear, disabling aggressive compiler optimizations _does_ improve debugging
experience, but it's not necessary.
[^separately]: That is what goes into those `-dbg` and `-debuginfo` packages if you ever wondered.

# How to use debuginfod servers

gdb (or another compatible debugger) will automatically try to fetch debug symbols from
the servers specified in the `DEBUGINFOD_URLS` environment variable. You can set it to
the address of the debuginfod instance provided by your Linux distro, or use the address
of the elfutils federated instance (`https://debuginfod.elfutils.org`) that will recursively
query other available debuginfod servers.

For example:

```text
~> DEBUGINFOD_URLS="https://debuginfod.elfutils.org" gdb /usr/bin/ls
Reading symbols from /usr/bin/ls...

This GDB supports auto-downloading debuginfo from the following URLs:
  <https://debuginfod.elfutils.org>
Enable debuginfod for this session? (y or [n]) y
Debuginfod has been enabled.
To make this setting permanent, add 'set debuginfod enabled on' to .gdbinit.
Downloading 499.77 K separate debug info for /usr/bin/ls
Reading symbols from /home/malor/.cache/debuginfod_client/d1f6561268de19201ceee260d3a4f6662e1e70dd/debuginfo...
```

Note that gdb tried to read debug symbols from the binary itself (`/usr/bin/ls`), did not find
any, and then suggested using the configured debuginfod server (`https://debuginfod.elfutils.org`).
The required DWARF information was successfully fetched and stored in the local cache to save
network bandwidth and speed up lookups for future debug sessions.

Attentive readers may wonder: how did gdb know _which_ debug symbols to request?

The answer is that gdb reads the "build id" value from the ELF note section:

```text
~> readelf --notes /usr/bin/ls

Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: d1f6561268de19201ceee260d3a4f6662e1e70dd
```

From [man 5 elf](https://man7.org/linux/man-pages/man5/elf.5.html):

> This section is used to hold an ID that uniquely identifies the contents of the ELF image.
Different files with the same build ID should contain the same executable content.

In practice, most frequently build id is a checksum computed over the generated content when
an ELF file is finalized by a linker. Because this value uniquely identifies a specific build,
it can be used to look up stripped debug symbols stored elsewhere, whether it's a local file or
a remote debuginfod service.

It goes without saying that anyone can run their own debuginfod server for the binaries they
build.

# Conclusion

debuginfod is a nice qualify of life improvement for debugging experience on Linux. Gone are
the days of searching for the correct package to install to get debug symbols, and Arch Linux
finally does not force you to build software from source! It's great to see debuginfod widely
adopted by the popular Linux distros and tools like gdb or valgrind.
