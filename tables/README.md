# ProFTPD Developer Guide: Module Tables

---

[Table of Contents](../toc.md)

---

## Module Tables

There are four _tables_ which act as the "glue" between the core ProFTPD
engine and a module.  None of the tables are _required_; however, having none
would make the module fairly useless.

Each module can define up to three different _handler tables_, which themselves
are registered via the _module table_.  These need not all be supplied.

They are grouped by the _type_ of handler:

* [`conftable`](configuration.md) lists all the configuration directive
  handler functions for the module.
* [`cmdtable`](command.md) lists all the FTP command handler functions for the
  module, including `PRE_CMD`, `POST_CMD`, and `LOG_CMD` handlers.
* [`authtable`](authentication.md) lists alternative authentication functions
  to be used, overriding the system-provided functions.

The fourth table, the [`module`](module.md) table, **is required** to be
defined by a module; indeed, this table/structure **is** the module definition.

---

[Table of Contents](../toc.md)

---
