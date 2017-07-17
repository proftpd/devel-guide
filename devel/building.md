# ProFTPD Developer Guide: Module Development: Building

---

[Table of Contents](../toc.md)

---

## Building Modules

An important part of module development is, of course, knowing how to get
your new module _built_ as part of the ProFTPD build process, by the build
system.

### The `contrib/` Area

All third-party (_i.e._ non-core) modules live under the `contrib/` directory
of the ProFTPD source tree.  In order to have your module built when compiling
ProFTPD, place your module source file, _e.g._ `mod_shiny.c`, into the
`contrib/` directory, and then make sure that module name _without_ the `.c`
extension ("mod_shiny" in this case) appears in the `--with-modules` option of
your `configure` command:

```
  $ ./configure --with-modules=mod_shiny
```

The `configure` script does the rest.

It is possible to split your module source into a header and a source file.
For most modules, this is not necessary, but if the module provides its
own API, like `mod_sql` or `mod_quotatab`, then having a separate header file
is a good design decision.  The `configure` script will automatically handle a
header file of the same name as the module, for example `mod_sql.c` and
`mod_sql.h`.

The `configure` script does its work by creating symlinks from the module
source files into the `src/` and `include/` directories, and listing the
object file names in the proper targets in the top-level Makefile.

### Third-Party Libraries

Sometimes your module may require additional libraries.  The `mod_ldap`,
`mod_sql_mysql`, `mod_sql_postgres`, and `mod_tls` modules are examples.
These modules indicate to the build system the additional third-party
libraries by including the following line near the top of the module source
file, in a comment:

```
  $Libraries: _flags_$
```

For example, `mod_ldap` requires that the linker use the `libldap` and
`liblber` libraries, so its line looks like:

```
  /*
   * $Libraries: -lldap -llber$
   */
```

### Additional Configuration

In rare cases, your module itself may require Autoconf support by having its
own `configure` script.  ProFTPD's build system does support per-module build
scripts, but _only_ if the module source code is in the `contrib/` area.
Support for per-module `configure` scripts by the `prxs` build tool does not
yet exist, although such support is indeed planned.

---

[Table of Contents](../toc.md)

---
