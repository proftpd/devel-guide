# ProFTPD Developer Guide: Handlers: Responses

---

[Table of Contents](../toc.md)

---

## Responding To Clients

Whenever the server processes a command from the client, it must generate an
appropriate, RFC-compliant response back to the client.  There are two main
ways of generating these responses, both usually done from within a command
handler.

### Response Chains

The first, and prefered, method of transmitting numeric-plus-text message
responses back to clients is via the internal _response chains_.  Using
these allows all handlers to add their own individual responses, which will
then all be sent _en masse_ after the command successfully completes or fails.

The core engine maintains _two_ such response chains, one chain for success
messages, and one for error messages.  When all handlers for a given command
have been called, the engine evaluates the final condition of the command.  If
it has failed (indicated by a handler returning `ERROR`), all responses
pending in the _error_ response chain are sent.  On the other hand, if the
command has completed _successfully_, the _success_ response chain is sent.
If a command has neither completed successfully _nor_ resulted in an error, it
is assumed to be an invalid or unsupported command, and the client is informed
approppriately.

In any case, once the "lifetime" of the command is over, and one (or none) of
the response chains has been sent, _both_ chains are destroyed.

The response chain method uses the following functions:

* `pr_response_add()`
* `pr_response_add_err()`

As you can see, with the response chain method, there are no multi-line
functions for multi-line replies, as there are with the second response
method. This is because the server automatically generates a multi-line
formatted response if it detects that there are two or more responses with the
same numeric response code waiting to be sent to the client.  In many cases,
you may be writing a handler, a `POST_CMD` phase handler for example, that
neither specifically knows nor cares what response codes other handlers are
using during _their_ responses.  In this case, you can use the special
`R_DUP` response, which tells the server that your response should be
_assumed_ to be part of a multi-line response started by another handler.

For example, assume the following `pr_response_add()` calls were made from
different modules.  Module "A" uses:

```
  pr_response_add(R_200, "Command successfully completed");
```

Later, module "B" then calls:

```
  pr_response_add(R_DUP, "Statistics for command '%s': none", cmd->argv[0]);
```

And then later, module "C" uses:

```
  pr_response_add(R_DUP, "XFOO post_cmd handler ran");
```

The final response sent to the client, assuming the command was successfully
handled, would be:

```
  200-Command successfully completed.
   Statistics for command 'XFOO': none.
  200 XFOO post_cmd handler ran.
```

See [include/ftp.h](https://github.com/proftpd/proftpd/blob/master/include/ftp.h) for the full listing of response code macros.

Responses to the client should be added using `pr_response_add()` when the
handler is going to return `HANDLED` or `DECLINED`; the `pr_response_add_err()`
function should be called if the handler is going to return `ERROR`.  At least
one response in the chain, either _success_ or _error_ chain, should be present
in order to return an RFC-compliant response code to the client.  **This
is important**; returning the wrong response code for a command, or no code at
all, can hang the client.  Note also that responses may be added to _both_
chains simultaneously, however only one chain of response will be flushed
back to the client.

### Direct Responses

The second way of sending responses to clients, the <i>direct response</i>
method, is incompatible with other handlers, and should be used only if the
handler is about to close the connection to the client, and terminate the
child process handling that connection.  This method **must** use one of the
functions listed below:

* `pr_response_send()`
* `pr_response_send_async()`
* `pr_response_send_ml()`
* `pr_response_send_ml_end()`
* `pr_response_send_ml_start()`

## Response Macros

The following are some macros specific to response generation, both by
modules and by the core engine:

### `PR_HANDLED`

```
  PR_HANDLED(cmd_rec *cmd)
```

Indicates that the handler has properly handled the command, and that the
engine should consider the command _completed_ and to continue processing with
the next phase.  Note that if a module handler returns `PR_HANDLED`, no
additional `CMD` phase handlers will be called, and instead the core engine
will start invoking the `POST_CMD` handlers for the command.

### `PR_DECLINED`

```
  PR_DECLINED(cmd_rec *cmd)
```

Indicates that the engine should act as though the handler was never called and
to continue processing.  `POST_CMD`-phase handlers _may_ be called after
this phase.

### `PR_ERROR`

```
  PR_ERROR(cmd_rec *cmd)
```

Indicates that a protocol or processing error occurred. `POST_CMD` handlers
will **not** be called after this phase; `POST_CMD_ERR` handlers _may_ be
called.

### `PR_ERROR_INT`

```
  PR_ERROR_INT(cmd_rec *cmd, int err_code)
```

Similar to `PR_ERROR`, except that the single integer numeric error is
indicated.  This is used **only** by authentication handlers, _e.g._ by
the `mod_ldap` and `mod_auth_unix` modules.

### `PR_ERROR_MSG`

```
  PR_ERROR_MSG(cmd_rec *cmd, int err_code, const char *err_text)
```

Similar to `PR_ERROR`, except that a string composed of _err_code_ and
_err_text_ is sent to the client.  In the case of configuration directive
handlers, the numeric argument is ignored, and the text message is displayed
to the user/sysadmin.

---

[Table of Contents](../toc.md)

---
