# ProFTPD Developer Guide: Internals: Restarts

---

[Table of Contents](../toc.md)

---

## Restarts

There are some tasks that should be done every time the daemon is restarted,
as when it receives a `SIGHUP` signal, tasks such as destroying and
re-allocating a module-specific `pool`, or "bouncing" any open file
descriptors (_e.g._ for log files) by closing and reopening them, _etc_.
A module can have arbitrary code executed as part of the restart process by
registering a callback function using the following registration
function:

```
  static void my_restart_ev(const void *event_data, void *user_data {
    ...
    /* Handle the restart event here. */
    ...
  }

  ...
    pr_event_register("core.start", my_restart_ev, NULL);
```

_**Do not**_ re-register restart event listeners.  Register them only _once_,
during module initialization.  Otherwise, you will bloat up the memory usage
of the daemon process, every time it is restarted.

---

[Table of Contents](../toc.md)

---
