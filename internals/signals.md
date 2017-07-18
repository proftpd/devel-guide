# ProFTPD Developer Guide: Internals: Signals

---

[Table of Contents](../toc.md)

---

## Signals

As of ProFTPD 1.2.6rc1, the way in which the daemon processes POSIX signals
changed.  Signal handlers are notorious for introducing race conditions
(see [here](http://lcamtuf.coredump.cx/signals.txt)); it is a consequence of
running code asynchronously, as signal handlers do.  The core engine was
significantly changed to avoid the actual processing of signals so that such
processing is done _synchronously_.  This means that a signal handler merely
sets a bit in bitset, and return swiftly, when a signal is delivered to the
process.  A call to the core signal processing function, `pr_signals_handle()`,
is done _frequently_ in order to act, synchronously, on any received signals.

This also means that signals do not, for the most part, interrupt the core
code.  For example, a `while()` loop that can potentially loop endlessly, and
which does not call `pr_signals_handle()`, will not be interrupted by most
signals (`SIGKILL` being an obvious exception).  Please keep this in mind when
developing your module; feel free to call `pr_signals_handle()` often,
_especially_ if a system or library call has been interrupted by a signal,
_i.e._ the call returns `-1`, and `errno` is `EINTR`, or when writing long
loops or recursive functions.  Here is an example that can be seen often
in the core code:

```
  while (func() < 0) {
    if (errno == EINTR) {
      pr_signals_handle();
      continue;
    }

    break;
  }
```

Alternatively, one could block signals before invoking the system call, and
unblock signals after the call, to prevent such interruptions.  Keep in mind
that any signals received while they are masked are silently ignored; they are
not queued by the kernel and redelivered to the process once they are unmasked.
A process can use the `sigpending(2)` system call to examine pending signals
(_i.e._ signals that have been raised while blocked), but this is different
from queueing; a process can see _which_ signals are pending, but not _how
many_ of each particular signal have been raised.  The lack of kernel-level
support for queueing of blocked signals is, I suspect, a design decision,
possibly because of the possibility of having to queue an infinite number of
signals.

### Signal Blocking and Masking

The following functions operate on the signals `TERM`, `CHLD`, `USR1`, `INT`,
`QUIT`, `ALRM`, `IO`, `BUS` and `HUP`:

* `pr_signals_block()`
* `pr_signals_unblock()`

For blocking _only_ `SIGALRM`, use these two functions:

* `pr_alarms_block()`
* `pr_alarms_unblock()`

The `SIGKILL` signal **cannot** be caught or ignored.

### Signal Handlers

The core engine implements handlers for the following signals:

* `SIGABRT`: Causes the daemon to abort.  This signal is handled by the
  daemon process; session processes retain the default handling.
* `SIGCHLD`: Cleanup an ended session.  This signal is handled by daemon
  process; session processes retain the default handling.
* `SIGUSR`: Disconnects all sessions.  This signal is handled by _session_
  processes; the daemon process retains default handling.
* `SIGHUP`: Restarts the daemon process and does not disconnect current
  sessions.  This signal is handled by the daemon process; it is ignored by
  session processes.
* `SIGTERM`: Perform a normal shutdown of the daemon.  It is handled by the
  daemon process; session processes retain default handling.
* `SIGXCPU`: [`RLimitCPU`](http://www.proftpd.org/docs/modules/mod_rlimit.html#RLimitCPU) reached; handled by the `SIGTERM` handler.
* `SIGINT`: Handled by the `SIGTERM` handler in both daemon and session
  processes.
* `SIGILL`: Handled by the `SIGTERM` handler in both daemon and session
  processes.
* `SIGFPE`: Handled by the `SIGTERM` handler in both daemon and session
  processes.
* `SIGSEGV`: Handled by the `SIGTERM` handler in both daemon and session
  processes.
* `SIGURG`: Ignored in both daemon and session processes.
* `SIGSTKFLT`: If defined, this signal is handled by the `SIGTERM` handler in
  both daemon and session processes.
* `SIGBUS`: If defined, this signal is handled by the `SIGTERM` handler in
  both daemon and session processes.
* `SIGIO`: Ignored in both daemon and session processes.

---

[Table of Contents](../toc.md)

---
