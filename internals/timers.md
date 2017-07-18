# ProFTPD Developer Guide: Internals: Timers

---

[Table of Contents](../toc.md)

---

## Timers

The core engine provides a mechanism for setting timers, which can be used
to set time limits on actions, or to perform actions on a recurring basis.

The Timers API is defined in [include/timers.h](https://github.com/proftpd/proftpd/blob/master/include/timers.h); the main functions are:

* `pr_timer_add()`
* `pr_timer_remove()`
* `pr_timer_reset()`

For the `pr_timer_add()` function arguments, _seconds_ is the length of the
timer, _timerno_ is an arbitrary timer ID, _m_ is a pointer to the module
registering the timer, and _cb_ is the function to be invoked once the timer
has expired.  If _timerno_ is `-1`, the timer will be assigned a "dynamic"
timer number greater than 1024.  This "dynamic" number is not random; rather,
it starts at 1024, as is incremented for every timer created with a _timerno_
of `-1`.  This function returns the given _timerno_, and has no error return
value.

The `pr_timer_remove()` function does just that, removing every timer with the
given ID _timerno_ registered by the module.  For this function, and for
`pr_timer_reset()`, the macro `ANY_MODULE` may be given as the module, in
which case _every_ timer with the given _timerno_ will be removed.  If a
matching timer is found, its _timerno_ is returned, otherwise `0` is returned.

The `pr_timer_reset()` function is also pretty self-explanatory, resetting the
timer with the given _timerno_ from a module.  If a matching timer is found,
the _timerno_ of the timer is returned, otherwise `0` is returned.

### Blocking Alarms

The `pr_alarms_block()` function is used to prevent timers from firing.  Timers
that would have fired while blocked are noted, and will be fired once alarms
are unblocked.  This ability can be used in critical sections of code which
cannot stand interruption.

As it notes in the comments in [src/timers.c](https://github.com/proftpd/proftpd/blob/master/src/timers.c), the blocking of alarms is done manually, rather
than via system calls, to allow for easier signal handling, portability,
detection of the number of blocked alarms, and to allow for nested
`block`/`unblock` calls.

The `pr_alarms_unblock()` function allows alarms to once again be delivered.
Any pending alarms that should have been delivered while alarms were blocked
are handled.

### Example Timer Callback

This excerpt from [modules/mod_auth.c](https://github.com/proftpd/proftpd/blob/master/modules/mod_auth.c) shows the [`TimeoutLogin`](http://www.proftpd.org/docs/modules/mod_auth.html#TimeoutLogin) timer implementation:

```
int login_timeout_cb(CALLBACK_FRAME) {
  /* Is this the proper behavior when timing out? */
  pr_response_send_async(R_421,
    "Login Timeout (%d seconds): closing control connection.", TimeoutLogin);
  
  pr_log_pri(PR_LOG_NOTICE, "%s", "FTP login timed out, disconnected.", 0, NULL);
  pr_session_disconnect(&auth_module, PR_SESS_DISCONNECT_TIMEOUT, "TimeoutLogin");
 
  /* Don't restart the timer */
  return 0;
}
```

The return value of the timer callback function is _**critical**_:

* If zero (`0`) is returned, the timer is done, and will not fire again.
* If non-zero, the timer is _restarted_, and will again be invoked once the
  configured length of time has elapsed.

The `CALLBACK_FRAME` macro is `#define`d to be:

```
  #define CALLBACK_FRAME          LPARAM p1,LPARAM p2,LPARAM p3,void *data
```

where `LPARAM` is `typedef`d to be:

```
  typedef unsigned long LPARAM;
```

Ostensibly, `LPARAM` is the way it is because it is the "longest bitsize
compatible with a scalar and largest pointer".  However, it does evoke
compiler warnings if pointers and data are passed this way.

Another thing of which to be aware is the use of the `CALLBACK_FRAME`
macro: it defines your function argument variable names _a priori_, which
can be easy to overlook when you are trying to figure out the name of the
variables to use in your timer callback function.  Unfortunately, most of the
current timer callback functions in the core code ignore the callback
variables.  When used, _p1_ will always be zero (with the current code),
_p2_ will be the `timerno` of the timer, _p3_ will be time
elapsed on the timer (which should be less than or equal to zero), and
_data_ will be `NULL`.

```
static int auth_sess_init(void) {
  /* Start the login timer */
  if (TimeoutLogin) {
    pr_timer_add(TimeoutLogin, TIMER_LOGIN, &auth_module, login_timer_cb);
  }

  ...

  return 0;
}
```

This use of `pr_timer_add()` illustrates how one registers one's timer
callback function, and starts the timer running.  Once the child process has
forked and `mod_auth`'s session initialization function called, the client has
until the timer expires to successfully authenticate.

When adding a timer, watch out for these predefined timer number/IDs:

```
  #define PR_TIMER_LOGIN               1
  #define PR_TIMER_IDLE                2
  #define PR_TIMER_NOXFER              3
  #define PR_TIMER_STALLED             4
```

There is no requirement that your timer number be a preprocessor macro.  The
author has successfully used randomly generated numbers as unique timer
identifiers, so that each timer could have custom actions and values.

### How Timers Work

The timer mechanism is based on `SIGALRM` and a _single_ signal handler for
that signal.  How then can multiple timers all use one signal handler?  A
linked list of timers is maintained, and that list is checked every time a
`SIGARLM` signal is received.  For each timer, the amount of time that has
expired is calculated, and if the specified timer duration has been reached,
that timer callback is invoked.

One caveat to keep in mind is that timers **do not** provide fine-grained time
resolution, and are not meant for precise work.  They do well for approximate
timing-based sorts of activities.

When `pr_timer_add()` is called, a "free list" of old, recyclable timers is
checked, to see if any are available.  If none are avaiable, a new timer is
allocated.  The timer is assigned the caller-provided parameters, then
inserted into the active timer list.

At present, the callback function is invoked thus (from [src/timers.c](https://github.com/proftpd/proftpd/blob/master/src/timers.c)):

```
  if (t->callback(t->interval, t->timerno, t->interval - t->count, t->mod) == 0 {
  ...
```

This shows what is actually passed as the arguments in the `callback_t`.

Be aware that the pending signal set for a process is _reset_ when `fork(2)`
is called.  This is why modules should and do register timers for the session
process in their respective session initialization functions.

---

[Table of Contents](../toc.md)

---
