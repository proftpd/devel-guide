# ProFTPD Developer Guide: Module Development: Client Disconnects

---

[Table of Contents](../toc.md)

---

## Client Disconnects

This refers to any special actions to be done on end-of-session/disconnects
that your module may wish to do.  Some modules may want to know if a client
explicitly disconnected via `QUIT` commands, or simply closed its connection.

Some protocols, like FTP, have an explicit `QUIT` command, so you might be
tempted to implement a `PRE_CMD` phase command handler for the `QUIT` command
in your module.

However, other protocols such as SSH/SFTP have no such explicit command.  In
addition, many clients (FTP and otherwise) do not _use_ such commands to
disconnect politely; they just close their TCP connection when they are done.

Given this, how can you know just _how_ a client disconnected?

For this, you should use the [`pr_session_get_disconnect_reason()`](https://github.com/proftpd/proftpd/blob/master/src/session.c#L156) function; see [here](https://github.com/proftpd/proftpd/blob/master/include/session.h#L28) for the list
of disconnect reason codes.  You can implement a listener for the `core.exit`
event to do this:

```
/* Custom exit event handler */
static void custom_exit_ev(const void *event_data, void *user_data) {
  /* We know that the client/connection is ending; use the disconnect reason to find out if
   * this was deliberate or not.
   */
  if (session.disconnect_reason == PR_SESS_DISCONNECT_CLIENT_QUIT) {
    /* Client told us to disconnect them */
  }
}
...

static custom_mod_init(void) {
  /* Register event handler */
  pr_event_register(&custom_module, "core.exit", custom_exit_ev, NULL);
}
```

The core ProFTPD engine will set the `session.disconnect_reason` value with the
reason for the disconnect, for logging purposes -- and for cases like this.

---

[Table of Contents](../toc.md)

---
