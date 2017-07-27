# ProFTPD Developer Guide: Module Development: Module Initialization

---

[Table of Contents](../toc.md)

---

## Module Initialization

There are two main points when a module can be _initialized_: during daemon
startup, and prior to handling a session.  In order to tell the core engine
that your module has code to be run at these points, you need to use the
initialization callback slots in the `module` structure at the bottom of your
module source file:

```
  module geewhiz_module = {
    ...

    /* Module initialization function */
    NULL,

    /* Session initialization function */
    NULL
  };
```

Rather than having `NULL`, you would have the name of your initialization
callback function there.  Both initialization callback functions have the same
prototype:

```
  int func(void);
```

What is the difference, then, between module initialization and session
initialization?  The former is called when the `proftpd` daemon starts, _before_
parsing the configuration file; the latter is called _after_ the daemon has
called `fork(2)` to create a new a process to handle a client, but before that
process handles any FTP commands sent by the client.  Module initialization,
if any is needed, tends to be simple: allocating module-specific resources
(_e.g._ memory pools, lists), perhaps initializing any third-party code
(_e.g._ libraries) used by the module, _etc_.  Most modules will make use more
of the session initialization function, which is where the module can act on
per-vhost/vhost-specific configuration settings, such as processing any of the
module's configuration directives that are in context of the server contacted
by the client.

The two above callbacks suffice for the majority of situations in which a
module developer might want to have initialization code executed.  However,
every now and then, there arises the case where a module may need to be
initialized _based on configured directives_, but still as part of the daemon
startup process.  Remember, the module initialization callback
is called **before** the configuration file is parsed, not after.  To
handle this scenario, use the "core.postparse" [event](../internals/events.md).
If a module wishes to have initialization code executed after the configuration
file has been parsed, then, in the module's module initialization function,
it will need to call `pr_event_register()` using the "core.postparse" event.

For example:

```
  static void geewhiz_postparse_ev(const void *event_data, void *user_data) {
    /* Do some possibly-configuration-influenced stuff here */
    ...

    return 0;
  }

  static int geewhiz_init(void) {
    ...

    /* Make sure we act on the needed directives' parameters */
    pr_event_register(&geewhiz_module, "core.postparse", geewhiz_postparse_ev,
      NULL);

    return 0;
  }

  module geewhiz_module = {
    ...

    /* Module initialization function */
    geewhiz_init,

    /* Session initialization function */
    NULL
  };
```

---

[Table of Contents](../toc.md)

---
