# ProFTPD Developer Guide: Module Development: Debugging

---

[Table of Contents](../toc.md)

---

## Debugging Modules

When developing a new module, server debugging can help pinpoint what is
happening in some cases; however, when tracking down such things as segfaults
("signal 11", `SIGSEGV`) or tracing the logic through a complex sequence of
function calls, more complicated forms of debugging are needed.

### Developer `configure` Options

By default, ProFTPD compiles into an executable suitable for running on a live
server.  However, this means that the executable is not as well suited for
debugging ;debug versions of programs often run slower.  To compile a debug
version of ProFTPD, use the `--enable-devel` option of the `configure` script.
By itself, this option has two side effects:

1. It enables quite a few `-W` compiler switches, which make the compiler more
   verbose and more picky about shadowed variables, prototype mismatches, use
   of `long double`, _etc_
1. The generated `proftpd` binary will *not* be stripped of its symbols when
   `make install` is run.  This means the binary will be larger, but it will
   also be easier to use standard debugging tools on the binary.  **Please**
   compile your module using `--enable-devel`, and fixing any generated errors
   and warnings, before contributing it.

The `--enable-devel` option can also take parameters that alter ProFTPD
behavior.  These parameters are:

* `coredump`: Enables `proftpd` to generate a `core` file under certain
  conditions (_e.g._ segfault, `SIGABRT`, _etc_).  Without this option,
  ProFTPD takes _stringent_ steps to explicitly **not** create `core` files,
  as they leak too much information.
* `coverage`: Enables the compiling of `proftpd` with code coverage support
  enabled; used for the [code coverage statistics](https://coveralls.io/github/proftpd/proftpd) for ProFTPD.
* `nodaemon`: Tells `proftpd` to not _daemonize_, but instead to always run
  in the foreground.  Debuggers often have issues when programs daemonize, as
  this process usually involves multiple `fork(2)` calls.
* `nofork`: Tells `proftpd` to **not** use `fork(2)` to create a new process
  for each connection; useful for debugging code paths only used during the
  handling of a session.  Newer versions of ProFTPD, however, provide the
  `-X` command-line option for this functionality, and do not require this
  build-time option.
* `profile`: Enables the compiling of `proftpd` with profiling support (see
  below).

These parameters can be combined when using the `--enable-devel` option, like
so:

```
  $ ./configure --enable-devel=nodaemon:nofork ...
```

or, to use them all:

```
  $ ./configure --enable-devel=coredump:nodaemon:nofork ...
```

### Tracing Programs

Your operating system should provide some utilities to help with system call
tracing (_e.g._ `strace` and `ltrace` on Linux, `ktrace` and `kdump` on
FreeBSD, `truss` on Solaris); read their respective man pages for more
information.

### GNU Debugger

The GNU debugger, `gdb`, is a superb tool for debugging programs.  Like many
such excellent tools, it can be intimidating for the uninitiated, but its use
is _highly_ recommended.  There are even graphical frontends,
such as [DDD](http://www.gnu.org/software/ddd/) for those who prefer graphical
environments.

`gdb` provides the most information when it is used to run a program that has
been linked with the debug version of `libc`, and whose symbols have not been
stripped.  Thus, build the `proftpd` executable using the `--enable-devel`
configure option.  Then, run your compiled `proftpd` executable like so:

```
  $ gdb ./proftpd
  Copyright 2001 Free Software Foundation, Inc.
  GDB is free software, covered by the GNU General Public License, and you are
  welcome to change it and/or distribute copies of it under certain conditions.
  Type "show copying" to see the conditions.
  There is absolutely no warranty for GDB.  Type "show warranty" for details.
  (no debugging symbols found)...
  (gdb)
```

At the prompt, start running the program, and pass it any of the usual
`proftpd` command-line options:

```
  (gdb) run -nd10
```

You can set breakpoints before running the program, step through certain
functions, _etc_.  One of the more helpful uses of `gdb` is for determining
where, and how, segfaults occur.  You run `proftpd` under the debugger until
you trigger the segfault.  `gdb` will display a message about catching the
`SIGSEGV` signal, and the name of the function, file, and line number at which
it happened.  Sometimes this segfault occurs within the C library, but is
almost always due to the application (_i.e._ module) code.  When this happens,
at the `(gdb)` prompt, generate a _backtrace_:

```
  (gdb) backtrace
```

A backtrace (or "stacktrace") is a listing of the function call stack, from
the start of the program to the point where the segfault occurs.  One can
then quickly narrow down where in the application code the bug is lurking.

### Memory Leaks

C programs are notorious for memory leaks and other memory allocation related
issues.  Numerous articles and programs have been written to address this.
I **highly** recommend using [Valgrind](http://valgrind.org/) for triaging
memory allocation (and corruption) issues.  For example:

```
  $ valgrind --leak-check=yes \
     --trace-children=yes \
     --log-file=/tmp/valgrind-proftpd.log \
     --num-callers=20 \
     /path/to/proftpd -X -nd10 -c /path/to/proftpd.conf
```

### Profiling

Profiling the code can be considered a form of debugging, once all other
functionality-related bugs have been fixed.  In order to compile a profile
version of `proftpd`, you need to use the `profile` parameter of
`--enable-devel`:

```
  $ ./configure --enable-devel=nodaemon:nofork:profile ...
```

The resulting daemon will only handle one session, as is necessary; the
daemonizing and `fork(2)` code interferes with the functioning of the
profiling code added by the compiler.  This profile version will generate two
files, `bb.out` and `gmon.out`, in the `PR_RUN_DIR` directory, which is
typically `/var/run/proftpd`.

Then, to see the call graph info, change to the directory where the `bb.out`
and `gmon.out` profiling files are, and invoke the following command:

```
  $ gprof /path/to/proftpd
```

The `gprof` program has quite a few options for generating call graphs and
timing information; read the man page to find out more information.

---

[Table of Contents](../toc.md)

---
