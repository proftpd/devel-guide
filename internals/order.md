# ProFTPD Developer Guide: Internals: Module Load Order

---

[Table of Contents](../toc.md)

---

## Module Load Order

Modules are prioritized and called in the _inverse_ order of which they were
loaded.  In other words, the _last_ loaded module is the **first** to receive
calls for a particular configuration directive, FTP command or authentication
request.

This can be used to allow later loaded modules (higher priority) to possibly
"override" early loaded (lower priority) modules.  The load order of modules
is thus _very_ important.

To see just what your module load order is, use:

    $ proftpd -l
    Compiled-in modules:
      mod_core.c
      mod_xfer.c
      mod_rlimit.c
      mod_auth_unix.c
      mod_auth_file.c
      mod_auth.c
      mod_ls.c
      mod_log.c
      mod_site.c
      mod_delay.c
      mod_facts.c
      mod_auth_pam.c
      mod_tls.c

The modules will be listed from lowest priority to highest priority.

Note, however, if that you use DSO/shared modules, then the module load order
is determined by the order of the
[`LoadModule`](http://www.proftpd.org/docs/modules/mod_dso.html#LoadModule)
directives in your configuration.  Again, the last module loaded via
`LoadModule` will be the _first_ module called.

---

[Table of Contents](../toc.md)

---
