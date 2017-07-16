# ProFTPD Developer Guide: Tables: Module Table

---

[Table of Contents](../toc.md)

---

## Module Tables

The `module` table defines the main interface between the module and the core
ProFTPD engine, providing pointers to the other handler tables, giving a name
to the module, _etc_.  The API version used by the module is denoted as well,
so that ProFTPD can recognize whether it is capable of using this module, or
whether the given API version is too old or too new.  As ProFTPD maintains an
internal list of modules in a doubly-linked list arrangement, the table defines
two pointers to the previous and next modules, which are set when ProFTPD
starts up; **do not** define these fields yourself.

Each module can supply up to two optional initialization callbacks via this
`module` table.  The first initialization callback is called immediately after
the module is loaded, while the second is called after a client connects to
ProFTPD, and the main ProFTPD daemon (if not in `inetd` mode) has forked off.

The purpose of this second initialization callback, called the session
initialization function, is to let the module perform any necessary
preparatory work once a client is connected and ProFTPD is ready to provide
service to the new client.  In `inetd` mode, the session initialization
function will be called immediately after ProFTPD is loaded, because ProFTPD
is **always** in "child mode" when run from `inetd/xinetd`.

Note that both of these initialization functions are _optional_.  If you do
not need them, simply set the function pointer to `NULL` in the `module` table.

### Example Module Table

```
module sample_module = {
  /* Always NULL */
  NULL, NULL,

  /* Module API version (current version: 2.0) */
  0x20,

  /* Module name */
  "sample",

  /* Module configuration handler table */
  sample_conftab,

  /* Module command handler table */
  sample_cmdtab,

  /* Module authentication handler table */
  NULL,

  /* Module initialization function */
  sample_init,

  <font color=green>/* Session initialization function */
  sample_sess_init

  /* Module version */
  "sample/1.0"
};
```

---

[Table of Contents](../toc.md)

---

