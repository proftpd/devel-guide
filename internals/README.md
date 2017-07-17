# ProFTPD Developer Guide: Internals

---

[Table of Contents](../toc.md)

---

## ProFTPD Internals

This is a collection of descriptions of some of the internal mechanisms and
workings of the core engine, such as how it decides which handlers to call
(and _when_), resolution of filesystem paths, _etc_.  I decided to put these
descriptions here for now, for lack of a better place for them; they are not
_exactly_ related to programming ProFTPD modules.

Please feel free to contribute explanations to this section.  The source for
ProFTPD is sadly lacking in in-source comments, and these descriptions aim to
fill the gap for aspiring developers.

Descriptions:

* [Process lifecycle](lifecycle.md)
* [Resource pools](pools.md)
* [Module load order](order.md)
* [Timers](timers.md)
* [Events](events.md)
* [Authenticating users](authenticating.md)
* [Configuration contexts](contexts.md)
* [Handling signals](signals.md)
* [Scoreboard](scoreboard.md)
* [Resolving <Directory> paths](directory.md)
* [Configuration "expressions"](expressions.md)
* [`.ftpaccess` files](ftpaccess.md)
* [Configuration parsing](parsing.md)
* [Symbol tables](symbols.md)
* [Restarts](restarts.md)
* [Schedules](schedules.md)
* [PAM](pam.md)

---

[Table of Contents](../toc.md)

---
