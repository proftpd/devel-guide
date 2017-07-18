# ProFTPD Developer Guide: Internals: Schedules

---

[Table of Contents](../toc.md)

---

## Schedules

The core engine provides a mechanism for setting _schedules_, which can be used
to perform actions on a recurring basis.  The triggering of schedules is
based on "loops", the number of times the daemon loops around while waiting
for incoming connections.  This means that schedules, rather than being based
on a clock, are based on server activity: a busy server loops more quickly
than a quiescent one.

The Schedule API to use when dealing with schedules is:

* `void schedule(void (*cb)(void *, void *, void *, void *), int nloops, void *a1, void *a2, void *a3, void *a4)`
*  `void run_schedule()`

The `schedule() function (which would ideally be named `pr_schedule_add()`)
registers a schedule callback function _cb_, which handles four `void *`
arguments, to be run every _nloops_.  When called, _cb_ will be passed the
registered pointers _a1_ through _a4_, which correspond to the four `void *`
arguments expected by the _cb_ function.

The `run_schedule()` function is called by the internal engine, and should
**not** be invoked anywhere else in the code without a very good reason.

### Example Schedule Callback

An excerpt from
<a href="../src/src/main.c.html#sig_rehash"><code>main.c</code></a>, this
defines the schedule callback for rehashing -- the registration of this
schedule occurs in the <code>SIGHUP</code> signal handler,
<a href="../src/src/main.c.html#sig_rehash"><code>sig_rehash()</code></a>:

```
  if (recvd_signal_flags & RECEIVED_SIG_RESTART) {
    recvd_signal_flags &= ~RECEIVED_SIG_RESTART;
    pr_trace_msg("signal", 9, "handling SIGHUP (signal %d)", SIGHUP);

    schedule(restart_daemon, 0, NULL, NULL, NULL, NULL);
  }
```

Here one sees how the re-processing of the configuration file, signalled via
`SIGHUP`, is scheduled:

```
  void restart_daemon(void *d1, void *d2, void *d3, void *d4) {
    /* restarting code here */
    ...

    return;
  }
```

In this example, the _nloops_ value given, `0`, means that that scheduled
callback will be run the next time `run_schedule()` is called.  The _nloops_
value is used to determine the frequency with which a schedule fires, assuming
it is meant to be a recurring schedule.  Which brings up a good point:
schedules are inherently _one-time triggers_.  They fire, then are removed
from the list of pending schedules.  Unlike [timers](timers.md), schedules
cannot re-register themselves simply by returning a certain value.  Instead,
if a recurring schedule is desired, simply have the callback function call
`schedule()` again, to add that callback back to the schedule list.

### How Schedules Work

The schedule mechanism is based on counters associated with each schedule
object, the _nloops_ member.  At certain points in the code, those counters
are decremented.  Once the counter reaches zero, the callback function for
that schedule object is triggered.  Schedule objects are registered and added
to an internally maintained list, and removed from that list once the callback
has triggered.

The _nloops_ counter is decremented by `run_schedule()`, which
is invoked at the following points in the core processing engine:

* `pr_netio_lingering_abort()`
* `pr_netio_lingering_close()`
* `pr_netio_poll()`
* `pr_netio_read()`
* `pr_netio_write()`
* `server_loop()`

Similar to timers, schedules **do not** provide fine-grained time resolution,
and are not meant for precise work.  In fact, the nature of the loops means
the time at which a schedule is triggered is very coarse.  Schedules are a 
mechanism for scheduling something to happen _at some point_ in the future,
when that activity is not time-critical.  Periodically checking state,
performing scrubbing or cleanup activities, these are activities suited for
scheduling.

---

[Table of Contents](../toc.md)

---
