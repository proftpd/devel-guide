# ProFTPD Developer Guide: Tables: Configuration Tables

---

[Table of Contents](../toc.md)

---

## Configuration Tables

The `conftable` structure declares configuration directives which this module
is interested in handling, and the handler functions for those directives.
The handlers will be invoked when the server parses configuration files either
during startup, during restarts, or when an `.ftpaccess` file is encountered.

Each entry in the configuration table includes the following fields, in this
order:

* configuration directive _name_
* configuration directive _handler_, a pointer to the handler function.  This
  function **must** have the signature/prototype:

```
  MODRET func(cmd_rec *)
```

### Example Configuration Table

```
static conftable sample_config[] = {
  { "FooBarDirective", add_foobardirective },
  { NULL }
};
```

The trailing `{ NULL }` _sentinel_ entry marks the end of the configuration
table, and should **always** be used.

---

[Table of Contents](../toc.md)

---
