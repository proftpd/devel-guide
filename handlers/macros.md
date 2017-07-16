# ProFTPD Developer Guide: Handlers: Macros

---

[Table of Contents](../toc.md)

---

## Configuration Directive Macros

### `CHECK_ARGS`

```
  CHECK_ARGS(cmd_rec *cmd, int expected_count)
```

Checks the number of parameters passed to the directive against the
`expected_count`.  Note that the count is _one less than_ `cmd_rec->argc`
because `cmd_rec->argc` includes `cmd_rec->argv[0]`, the directive name itself.
This check also ensures only that there is an equal _or greater_ number of
parameters than the number requested.  If this check fails, a generic error is
sent to the caller.

Example:

```
  CHECK_ARGS(cmd, 1);
```

### `CHECK_HASARGS`

```
  CHECK_HASARGS(cmd_rec *cmd, int expected_count)
```

Performs a Boolean check, to see if the number of parameters passed to the
directive matches the given count

Example:

```
  if (!CHECK_HASARGS(cmd, 3)) {
    ...
  }
```

### `CHECK_VARARGS`

```
  CHECK_VARARGS(cmd_rec *cmd, int min_argc, int max_argc)
```

If this check fails, a generic error is sent to the caller.

Example:

```
  CHECK_VARARGS(cmd, 1, 5);
```

### `CHECK_CONF`

```
  CHECK_CONF(cmd_rec *cmd, int flags)
```

Makes sure that this directive is not being used in the wrong section/context,
_i.e._ if the directive is only available or applicable inside certain
configuration sections.

In this example below, we are allowing the directive inside of `<Anonymous>`
and `<Limit>` sections, and not elsewhere.  If this checks fails, a generic
error is logged and the handler returns.

```
  CHECK_CONF(cmd, CONF_ANON|CONF_LIMIT);
```

### `CONF_ERROR`

```
  CONF_ERROR(cmd_rec *cmd, char *message)
```

Helper macro for constructing/formatting an error message to the user; the
constructed message will include the file name, line number, and configuration
directive involved.

Example:

```
  CONF_ERROR(cmd, "syntax: expected numbers, not letters");
```

## Command Macros

### `CHECK_CMD_ARGS`

```
  CHECK_CMD_ARGS(cmd_rec *cmd, int argc)
```

Checks that the number of arguments passed to the command is **exactly** the
indicated count.  If this check fails, a 501 response code, with a message
stating the problem, is sent to the client.

Example:

```
  CHECK_CMD_ARGS(cmd, 2);
```

### `CHECK_CMD_MIN_ARGS`

```
  CHECK_CMD_MIN_ARGS(cmd_rec *cmd, int argc)
```

Checks that the command has received _at least_ the indicated number of
arguments.  If this check fails, a 501 response code is sent to the client.

Example:

```
  CHECK_CMD_MIN_ARGS(cmd, 2);
```

## Authentication Macros

---

[Table of Contents](../toc.md)

---
