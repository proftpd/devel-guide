# ProFTPD Developer Guide: Handlers: Commands

---

[Table of Contents](../toc.md)

---

## Command Handlers

ProFTPD defines different _phases_ in the handling of a command.  There are
currently six such command phases, processed in the following order:

* `PRE_CMD`
* `CMD`
* `POST_CMD`, `POST_CMD_ERR`
* `LOG_CMD`, `LOG_CMD_ERR`

When ProFTPD receives a command from a client, it enters a "cascading" command
delivery process:

1. Call any [`PRE_CMD`](#pre-command-phase) handlers that match the special
   `C_ANY` wildcard.
1. Call any [`PRE_CMD`](#pre-command-phase) handlers that match the command
   _exactly_.
1. Call any [`CMD`](#command-phase) handlers that match the special `C_ANY`
   wildcard.
1. Call any [`CMD`](#command-phase) handlers that match the command _exactly_.
1. Call any [`POST_CMD`](#post-command-phase) handlers that match the special
   `C_ANY` wildcard, _if the `CMD` handlers succeeded_.
1. Call any [`POST_CMD`]#(post-command-phase) handlers that match the command
   _exactly_, _if the `CMD` handlers succeeded_.
1. Call any [`POST_CMD_ERR`](#post-command-error-phase) handlers that match the
   special `C_ANY` wildcard, _if the `PRE_CMD` or `CMD` handlers returned
   `ERROR`_.
1. Call any [`POST_CMD_ERR`](#post-command-error-phase) handlers that match the
   command _exactly_, _if the `PRE_CMD` or `CMD` handlers returned `ERROR`_.
1. Call any [`LOG_CMD`](#log-command-phase) handlers that match the special
   `C_ANY` wildcard, _if the `CMD` handlers succeeded_.
1. Call any [`LOG_CMD`](#log-command-phase) handlers that match the command
   _exactly_, _if the `CMD` handlers succeeded_.
1. Call any [`LOG_CMD_ERR`](#log-command-error-phase) handlers that match the
   special `C_ANY` wildcard, _if the `PRE_CMD` or `CMD` handlers returned
   `ERROR`_.
1. Call any [`LOG_CMD_ERR`](#log-command-error-phase) handlers that match the
   command _exactly_, _if the `PRE_CMD` or `CMD` handlers returned `ERROR`_.

The return value of a particular command handler can allow or disallow further
dispatching.  This allows an _earlier_ module (in the module load order) to
_overload_ a phase for a particular command, and allow or deny the command as
needed.

### Pre-Command Phase

As a general rule of thumb, `PRE_CMD` command handlers should perform validity
checks on the command (syntax, applicability, access control restrictions,
resource availability, _etc_).  When one of these handlers returns `DECLINED`,
the next `PRE_CMD` command handler will be called.  However, if `ERROR` is
returned, the dispatching to _all_ command handlers will stop completely, with
the exception of [`LOG_CMD_ERR`](#log-command-error-phase) handlers.  A
`PRE_CMD` handler should **never** return `HANDLED` itself; doing so **will**
cause bugs, as it breaks the semantics expected by the core engine's command
dispatching system.

#### Example `PRE_CMD` Handlers

Below is an example of a `PRE_CMD` handler which simply logs all received
commands using the `pr_log_debug()` function.  It is careful to return
`DECLINED`, otherwise other `PRE_CMD` handlers in the phase would not also
be called to process the command.  Note that in order for this to work
properly, this module would need to be loaded _last_, or after any other
modules which do not themselves return `DECLINED` for all of their `PRE_CMD`
command handlers.  In practice, you should **always** return `DECLINED`, unless
you need to deny/reject the command.

```
  MODRET pre_cmd(cmd_rec *cmd) {
    pr_log_debug(DEBUG0, "RECEIVED: command '%s', arguments '%s'",
      (char *) cmd->argv[0], cmd->arg);
    return PR_DECLINED(cmd);
  }
```

### Command Phase

`CMD` handlers should do the actual _**work**_.  Much like `PRE_CMD` handlers,
if `DECLINED` is returned by a `CMD` phase handler, then the next registered
`CMD` handler for the command will be called for the given `cmd_rec`.  And,
like `PRE_CMD` handlers, if `ERROR` is returned by the handler, then dispatching
to other command handlers stops, except for `LOG_CMD_ERR` phase handlers.

_Unlike_ `PRE_CMD` handlers, though, a `CMD` handler **is** expected to return
`HANDLED` if it successfully handled/processed the command.  `CMD` handlers
will be called, for the same command, until _one_ of them returns `HANDLED`
or `ERROR`.

### Post-Command Phase

`POST_CMD` handlers generally handle any kind of cleanup.  Should a command
pass through the relevant `PRE_CMD` and `CMD` phase handlers without error,
then the registered `POST_CMD` handlers will be called.  If these handlers
return `ERROR`, though, their response message, if any, will be **not** be
sent to the client, and instead will only be logged.  Not much else can be
done at that point, since the _real work_ of the command has already been
done by the previously-invoked `CMD` handlers.

As with `PRE_CMD` handlers, a `POST_CMD` handler should **never** return
`HANDLED`.

#### Example `POST_CMD` Handlers

The global variable `session` contains a lot of important data after a
file/directory transfer of any kind, and is not cleared until the `mod_xfer`
module has its `LOG_CMD` phase handlers called.  This fact is used in these
`POST_CMD` handlers, declared for the `LIST`, `NLST`, `RETR`, and `STOR`
FTP commands in order to calculate the total data transfer for a session.

```
  static off_t total_bytes_received = 0, total_bytes_xferred = 0;

  MODRET post_cmd_list(cmd_rec *cmd) {
    total_bytes_xferred += session.xfer.total_bytes;
    return PR_DECLINED(cmd);
  }

  MODRET post_cmd_nlst(cmd_rec *cmd) {
    return post_cmd_list(cmd);
  }

  MODRET post_cmd_retr(cmd_rec *cmd) {
    return post_cmd_list(cmd);
  }

  MODRET post_cmd_stor(cmd_rec *cmd) {
    total_bytes_received += session.xfer.total_bytes;
    return PR_DECLINED(cmd);
  }
```

### Post-Command Error Phase

<code>POST_CMD_ERR</code> type handlers handle any reporting and logging
of error conditions that may occur during the processing of a
<code>PRE_CMD</code> or <code>CMD</code> handler.  One thing that distinguishes
a <code>POST_CMD_ERR</code> handler from a <code>LOG_CMD_ERR</code> handler
is that any error responses added during a <code>POST_CMD_ERR</code> handle,
using <code>add_response_err()</code>, will be seen by the client when the
error response chain is flushed; similar responses added by a
<code>LOG_CMD_ERR</code> handler are not seen by the client.

#### Example `POST_CMD_ERR` Handlers

```
  MODRET post_stor_err(cmd_rec *cmd) {
    pr_response_add_err(R_DUP, "%s failed: try harder next time!",
      (char *) cmd->argv[0]);
    return PR_DECLINED(cmd);
  }
```

### Log Command Phase

`LOG_CMD` phase handlers handle logging of successful commands, and are
called only after the command has been completely processed successfully, and
the response sent to the client.

#### Example `LOG_CMD` Handlers

```
  MODRET log_cmd(cmd_rec *cmd) {
    pr_log_debug(DEBUG0, "SUCCESSFUL: command '%s', arguments '%s'",
      (char *) cmd->argv[0], cmd->arg);
    return PR_DECLINED(cmd);
  }
```

### Log Command Error Phase

And, as might be expected, `LOG_CMD_ERR` phase handlers handle logging of
failed commands.  `LOG_CMD_ERR` handlers will be called in the event of
`ERROR` being returned from either a `PRE_CMD` or a `CMD` handler for that
command.  At present, only the `mod_log` module provides a `LOG_CMD_ERR` phase
handler, and it uses the same handler for _both_ `LOG_CMD` _and_ `LOG_CMD_ERR`
handling.  If for any reason you wanted to handle logging or reporting of
failed commands, either all commands or just specific commands, this would be
the type of handler to use.

---

[Table of Contents](../toc.md)

---

