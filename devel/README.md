# ProFTPD Developer Guide: Module Development

---

[Table of Contents](../toc.md)

---

## Module Development

As with any software development, there are certain things to watch out for
when building a module.  There are also other things to consider as part of
the development process, such as the `configure` scripts and documentation.

If you are developing a module for wider distribution with ProFTPD, please
keep in mind the overall design philosophy: no use of `system(2)` or the
`exec` family of system calls.  Helper applications to accompany the module
are acceptable; the server (and modules) should simply not invoke any code
outside of its control.

Also, respect the normal filesystem permissions.  ProFTPD has always honored
the normal Unix permissions, and may provide access controls _in addition to_
those of the filesystem, but *does not* override them.  Overriding the
filesystem permissions only leads to trouble; it violates the [Principle of
Least Surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).

### Documentation

_**No**_ modules or patches will be accepted into the ProFTPD source **without
documentation**.  Modules submitted without documents on what they do, on how
to properly use any configuration directives the module might define, or on
how to install and use the module, are next to useless by the ProFTPD community.

Patches without descriptions of what they are fixing, _and why_, force
developers to read the code of the patch in order to decipher the purpose; the
author of the patch has the best insight into the patch's purpose.

### Common Tasks

As you develop your module, you will find that there are some routines and
operations needed that will have inevitably been done before, either in
other modules, or in the core code itself.  These pages describe some of the
common tasks you can expect to encounter, and how they can be done:

* [Building](building.md) your module
* [Debugging](debugging.md) your module
* Using third-party [libraries](libraries.md) in your module
* [Initializing](initializing.md) your module
* Using session [credentials](credentials.md) (_e.g._ user, groups, class)
* Adding new [`SITE`](site.md) commands with your module
* [Logging](logging.md) in your module
* [Chroot](chroot.md) workarounds in your module
* General [tips](tips.md) as you develop your module

### Licensing

For the most part, this is easy: GPLv2.  However, when working with
third-party libraries or code, be sure to include the necessary copyrights and
licensing information as required by the use of those files.  Some licensing
issues to keep in mind are mentioned here:

* <http://www.gnu.org/licenses/gpl-faq.html#GPLModuleLicense>
* <http://www.gnu.org/licenses/gpl-faq.html#GPLAndPlugins>

Due to the nature of the ProFTPD odule interface, third-party modules are
considered part of the main binary, and thus by definition fall under the GPL
(or [GPL-compatible licenses](http://www.gnu.org/licenses/license-list.html#GPLCompatibleLicenses)), unless explicitly excluded; the OpenSSL code is excluded
this way.

If developing modules that rely on cryptographic code, please be cognizant of
any governmental laws or restrictions on the cryptographic code in use,
in its licensing and distribution.

### Security

Always, _**always**_, **ALWAYS** check, verify, and sanitize user input, be it
from configuration files such as `.ftpaccess`, from the configuration
directive arguments for your modules, from any FTP commands you have created,
_etc_.  If you are going to open and write a file with root privileges, check
to see if the file exists first.  If it does, check whether it is a symlink,
has data, whatever.  **Do not** blindly write to a file name, unless you are
absolutely certain of the conditions of the directory and file to which you
are writing.

#### Secure Practices

Here are some recommended [resources](security.md) for further reading on
secure programming and practices to keep in mind.

---

[Table of Contents](../toc.md)

---
