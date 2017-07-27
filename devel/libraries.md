# ProFTPD Developer Guide: Module Development: Third-Party Libraries

---

[Table of Contents](../toc.md)

---

## Third-Party Libraries

This refers to any third-party libraries that your module might require, such
as the MySQL, LDAP, and OpenSSL libraries, or any other library.  Such
dependencies will require changes to the `configure` script which is run before
ProFTPD is compiled.

To have a library linkage (_e.g._ `-lcrypto`) _automatically_ added to the
Makefile, use the special `$Libraries$` keyword near the top of your module
source file:

```
  /* ----DO NOT EDIT BELOW THIS LINE----
   * $Libraries: -lcrypto$
   */
```

This covers the case where the library in question is common, and applies
to all platforms.  The ProFTPD build system scans module source files for
this `$Library$` keyword, just for such purposes.

If that library is in a non-standard location, a library path may need to
provided (_i.e._ the `-L` compiler/linker option).  This can be done at
`configure`-time using the `--with-libraries` option of the `configure` script:

```
  ./configure --with-libraries=/path/to/other/library ...
```

Alternatively, the `LDFLAGS` environment variable can be used to pass linker
information into the build process:

```
  ./configure LDFLAGS=linker-options ...
```

If use of `--with-libraries` or `LDFLAGS` is necessary when building your
module, make sure to include this requirement as part of your module
documentation.

---

[Table of Contents](../toc.md)

---
