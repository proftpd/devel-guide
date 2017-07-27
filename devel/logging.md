# ProFTPD Developer Guide: Module Development: Module Logging

---

[Table of Contents](../toc.md)

---

## Module Logging

Logging is an important part of any module.  It helps the administrator keep
track of what the module is doing, and of any problems that might be causing
trouble.

The first step is to `#define` a label for your module, comprised of the
module name _and_ its version.  For example:

```
  #define MOD_SHINY_VERSION  "mod_shiny/1.1.0"
```

This identifying label helps to determine the version of your module, should
someone need to check what version they are using.  It is also an extremely
helpful part of the logging process.

### System Logging

When developing a module, please make use of the `pr_log_pri()` and
`pr_log_debug()` functions, where appropriate.  Not only will this aid you as
your module develops, but it will help users report potential problems once
your module is released.

When calling these functions, make use of the `#define`d module label, _e.g._:

```
  pr_log_pri(PR_LOG_ERR, MOD_WRAP2_VERSION ": error: unable to open '%s': %s", path, strerror(errno));

  pr_log_debug(DEBUG4, MOD_WRAP2_VERSION ": checking ACLs for user %s", session.user));
```

With statements like this, the output from your module is much more easily
identified in the system/server log files.  Not all third-party modules do this,
it is true; it is a good habit to adopt, though.

### Per-Module Log Files

For some modules, it makes more sense to have them log to their own separate
log files, rather than adding the module-specific output to the rest of
the ProFTPD logging.  Use the `pr_log_openfile()` and `pr_log_writefile()`
functions for this use case, as these functions handle such things as
checking for world-writable directories (logs should _not_ be accessible to the
world, nor _writable_ by the world) or symlinks (which are subject to race
conditions).

### Trace Logging

In addition to any system and/or per-module logging your module may do,
you will probably find it useful to use
[_trace logging_](http://www.proftpd.org/docs/howto/Tracing.html) (or just
"tracing").

Use of a trace channel named after your module is often used to provide _very_
fine-grained logging.   One major benefit of tracing is that it can be
dynamically set/tuned, on a running server, via [`ftpdctl trace`](http://www.proftpd.org/docs/contrib/mod_ctrls_admin.html#trace).

---

[Table of Contents](../toc.md)

---
