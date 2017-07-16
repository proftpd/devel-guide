# ProFTPD Developer Guide: Tables: Command Tables

---

[Table of Contents](../toc.md)

---

## Command Tables

The `cmdtable` is a structure which lists all FTP command handlers for the
module.  Each entry in this table contains a special command _phase_ field
which determines phase of handler, be it `PRE_CMD`, `CMD`, `POST_CMD`, or
`LOG_CMD` (see the [command](../handlers/command.md) handlers documentation
for more details on these phases).

Each entry in the command table contains the following fields, in this order:

* command _phase_
* command _name_, or the actual `NULL`-terminated ASCII text sent by a client,
  _in uppercase_, for this command.  The
  [include/ftp.h](https://github.com/proftpd/proftpd/blob/master/include/ftp.h)
  header file has macros which cover all of the known RFC-defined FTP protocol
  commands.  This _name_ field can also be the special macro `C_ANY`, which
  covers **all** commands.
* command _group_, which is the access control group, referenced by `<Limit>`
  directives, to which this command belongs.  This group can be either
  `G_DIRS` for commands related to directories, `G_READ` for commands related
  to read operations (such as reading/downloading files), `G_WRITE` for
  commands related to write or modify operations (such as writing/uploading
  files), or the special `G_NONE` value for those commands against which a
  `<Limit READ WRITE DIRS>` will not be applied.
* command _handler_, a function pointer to the handler function for this
  command.  This function **must** have the signature/prototype:

```
  MODRET func(cmd_rec *cmd)
```

* _requires authentication_, a Boolean value.  Set this to `TRUE` if the
  command **cannot** be used before authentication has succeeded, `FALSE`
  if the command can be used at any time after the client connects.
* _while transferring_, a Boolean value. **Note**: as of ProFTPD 1.1.5,
  this field is _obsolete_ and is no longer used.

### Example Command Table

This command table is from the `mod_sample.c` module:

```
cmdtable sample_commands[] = {
  { PRE_CMD,  C_ANY,  G_NONE, pre_cmd,        FALSE, FALSE },
  { LOG_CMD,  C_ANY,  G_NONE, log_cmd,        FALSE, FALSE },
  { POST_CMD, C_RETR, G_NONE, post_cmd_retr,  FALSE, FALSE },
  { POST_CMD, C_STOR, G_NONE, post_cmd_stor,  FALSE, FALSE },
  { POST_CMD, C_APPE, G_NONE, post_cmd_stor,  FALSE, FALSE },
  { POST_CMD, C_LIST, G_NONE, post_cmd_list,  FALSE, FALSE },
  { POST_CMD, C_NLST, G_NONE, post_cmd_nlst,  FALSE, FALSE },
  { CMD,      "XFOO", G_DIRS, cmd_xfoo,       TRUE,  FALSE },
  { 0, NULL }
};
```

The one entry in the example table that defines a `CMD` entry demonstrates
how to declare a command handler function for a _custom_ FTP command, `XFOO`
in this case:

```
  { CMD,      "XFOO", G_DIRS, cmd_xfoo,       TRUE,  FALSE },
```

Most FTP clients allow the user to send arbitrary commands to the FTP server;
this ability, combined with custom FTP commands handlers such as this `XFOO`,
allow the programmer to create arbitrary, albeit not widely implemented, FTP
commands.

The trailing `{ 0, NULL }` _sentinel_ entry marks the end of the command table,
and should **always** be used.

---

[Table of Contents](../toc.md)

---
