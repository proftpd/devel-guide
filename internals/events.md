# ProFTPD Developer Guide: Internals: Events

---

[Table of Contents](../toc.md)

---

## Events

### What are Events?

Events are an intra-process notification system, used to indicate certain
conditions to other parts of the code that have registered an interest in
those conditions.  A module can register a _listener_ callback for any given
event; when that event is _generated_, the callback is invoked.  All
registered listeners are invoked in the usual [module load order](order.md).

The Events API replaces the old registration functions that were specifically
for process exits, daemon startup, and configuration parsing.  That old system
would not scale well, as several new functions would need to be added to the
core API for every new type of event.  The new Events API can handle any
number of arbitrary events; modules can easily generate their own custom
events without requiring code changes in the core APIs.

### When *Not* To Use Events

The ProFTPD API contains many hooks and entry points into the
[lifecycle](lifecycle.md) of an FTP session: authentication handlers, command
handlers, configuration directive handlers, filesystem and network callbacks,
_etc_.  These APIs cover most of the situations in which a module developer is
interested.  Events are **not needed** in cases where the existing API can be
used to achieve the same goal.

### List of Common Events

Some of the current events generated by ProFTPD are:

* `core.create-home`: Generated when the
   [`CreateHome`](http://www.proftpd.org/docs/modules/mod_auth.html#CreateHome)
   directive is used, and a home directory is actually being created for the
   user who is authenticating.  The name of the user whose home is being
   created is sent as the `event_data`.
* `core.exit`: Generated when the session process [exits](exits.md).
* `core.max-connection-rate`: Generated when the [`MaxConnectionRate`](http://www.proftpd.org/docs/modules/mod_core.html#MaxConnectionRate)
  limit, if present, is reached.
* `core.max-instances`: Generated when the [`MaxInstances`](http://www.proftpd.org/docs/modules/mod_core.html#MaxInstances)
  limit, if present, is reached.
* `core.module-load`: Generated after a module is loaded.  The `event_data`
  for this event is the name of the module that was loaded, _e.g._
  "mod_example.c".
* `core.module-unload`: Generated after a module is unloaded.  The `event_data`
  for this event is the name of the module that was loaded, _e.g._
  "mod_example.c".
* `core.preparse`: Generated just before the core engine parses the
  configuration files.  **Note:** A handler for this event **must** be
  registered in a module initialization function, and **not** in a session
  initialization function, in order to be used.  The "core.preparse" event is
  generated _before_ starting any sessions.
* `core.postparse`: Generated just after the core engine parses the
  configuration files.  **Note:** A handler for this event **must** be
  registered in a module initialization function, and **not** in a session
  initialization function, in order to be used.  The "core.postparse" event is
  generated _before_ starting any sessions.
* `core.restart`: Generated just before the core engine processes the
  `SIGHUP` signal.  There are some tasks that should be done every time the
  daemon is restarted via the `SIGHUP` signal, tasks such as destroying and
  reallocating a module-specific `pool`, or "bouncing" any open file
  descriptors, such as log files, by closing and reopening them, _etc_.
   _**Do not**_ re-register "core.restart" listeners; register them _only once_,
  during module initialization.  Otherwise, you will bloat up the memory usage
  of the daemon process, every time it is restarted.
* `core.startup`: Generated by the core engine after it has "daemonized".
  **Note:** A handler for this event **must** be registered in a module
  initialization function, and **not** in a session initialization function,
  in order to be used.  The "core.startup" event is generated _before_ starting
  any sessions.
* `mod_auth.anon-reject-passwords`: Generated by the `mod_auth` module when
  the [`AnonRejectPasswords`](http://www.proftpd.org/docs/modules/mod_auth.html#AnonRejectPasswords) rule is triggered.
* `mod_auth.max-clients`: Generated by the `mod_auth` module when the
  [`MaxClients`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxClients)
  limit, if present, is reached.
* `mod_auth.max-clients-per-class`: Generated by the `mod_auth` module when the
  [`MaxClientsPerClass`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxClientsPerClass)
  limit, if present, is reached.
* `mod_auth.max-clients-per-host`: Generated by the `mod_auth` module when the
  [`MaxClientsPerHost`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxClientsPerHost)
  limit, if present, is reached.
* `mod_auth.max-clients-per-user`: Generated by the `mod_auth` module when the
  [`MaxClientsPerUser`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxClientsPerUser)
  limit, if present, is reached.
* `mod_auth.max-hosts-per-user`: Generated by the `mod_auth` module when the
  [`MaxHostsPerUser`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxHostsPerUser)
  limit, if present, is reached.
* `mod_auth.max-login-attempts`: Generated by the `mod_auth` module when the
  [`MaxLoginAttempts`](http://www.proftpd.org/docs/modules/mod_auth.html#MaxLoginAttempts)
  limit, if present, is reached.

Note that if a non-`NULL` `user_data` pointer is provided when registering an
event listener, and that pointer is allocated from a pool that does not
survive a restart, then the caller should unregister that event listener
during a restart, and re-register the listener again with an updated pointer.
The problem is that the Events API keeps a pointer to the given `user_data`,
and when the pool from which that data is destroyed, the pointer becomes
stale/invalid.  The invalid pointer would be passed to the event listener,
after a restart, unless the pointer is properly replaced via re-registration.

### Example Event Code

```
  /* Event handler for the 'mod_foo.foo' event. *
  static void my_foo_ev(const void *event_data, void *user_data) {
    (void) pr_log_writefile(my_logfd, MOD_MY_VERSION,
      "the 'mod_foo.foo' event occurred");
    return;
  }

  ...

    /* Register our callback for the 'mod_foo.foo' event.
     * The last NULL argument indicates that we are not interested in passing
     * our own "user_data" parameter to the callback.
     */
    pr_event_register(&my_module, "mod_foo.foo", my_foo_ev, NULL);

  ...

    /* Meanwhile, back in mod_foo.c's handlers...*/
    pr_event_generate("mod_foo.foo", NULL);
```

Events can be generated for which there are no listeners registered, and
listeners can be registered for events which are never generated.

---

[Table of Contents](../toc.md)

---