# ProFTPD Developer Guide: Internals: Exits

---

[Table of Contents](../toc.md)

---

## Handling Process Exits

Sometimes, a module developer may wish to have a bit of code executed when
an FTP session ends, or when the daemon is being shut down.  The ProFTPD API
allows for this by providing the following two events:

* "core.exit" for session exits
* "core.shutdown" for daemon shutdown

The module developer then registers an [event](events.md) listener, a callback,
for these events.

All such registered callbacks will be executed, in their order of registration,
as part of the exit process for the session, or for the daemon.  Note that
the "core.shutdown" listeners will be executed with root privileges.  This
may _not_ be true for "core.exit" listeners, as often root privileges are
dropped completely for the session process.

Exit event listeners, as mentioned, are run as part of the exit process, which
is executed according to various conditions: an ACL denies access to the client,
a `MaxClients` or related limit is reached, the session times out, the client
sent a `QUIT` command, _etc_.  For whatever reasons, the session process will
decide to end the session, and will call `pr_session_disconnect()`.
This function will end up calling `pr_session_end()`, and that will call the
internal `sess_cleanup()` function.  This _session cleanup_ function performs
the following actions:

* Deletes the session entry from the [scoreboard](scoreboard.md)
* Generates the "core.exit" [event](events.md)
* Cleans up the `wtmp` log
* Closes any _lingering_ network connections

And finally, `sess_cleanup()` causes the process to end via the `_exit(2)`
system call.

_**Do not**_ re-register listeners for exit events; register them only _once_,
during module (or session) initialization.  Mulitple registration will cause
your listener to be executed multiple times, which is probably not what is
intended.  Also, take care when writing exit event listeners.  The listener
should not **assume** anything about the state of pointers or variables: check
that pointers are not `NULL` before dereferencing them, that values are within
acceptable ranges before operating on them, _etc_.  Failure to do so will cause
your exit listener to segfault.

---

[Table of Contents](../toc.md)

---
