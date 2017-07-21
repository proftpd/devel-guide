# ProFTPD Developer Guide: Internals: Configuration Contexts

---

[Table of Contents](../toc.md)

---

## Configuration Contexts

At present, ProFTPD has seven different configuration sections or _contexts_:

* "server config"
* `<Anonymous>`
* `<Directory>`
* `<Global>`
* `<Limit>`
* `<VirtualHost>`
* `.ftpaccess`

These contexts are determined, in configuration handlers, using the
`CHECK_CONF` macro.

### The "server config" Context

The "server config" context encompasses everything _outside of_ the other
contexts (_i.e._ every configuration directive that is not explicitly
contained within another configuration context), and indicated by the value
`CONF_ROOT` in a configuration directive handler.

### The `<Anonymous>` Context

The `<Anonymous>` section is used to set up the very common configuration of
an anonymous FTP server.  It does a `chroot(2)` to the anonymous FTP directory
by default, and turns off the requirement for a valid password, requesting only
a valid email address as the password.  Other system binaries or files need
not be contained within the `<Anonymous>` directory.

The `CONF_ANON` value is used to indicate the `<Anonymous>` context.  Since
an `<Anonymous>` section is not considered a separate server, but rather a
_subset_ of its containing server, any configuration directives set for that
server _will_ be in effect for the `<Anonymous>` section as well, unless
overridden by a directive of the same name _within_ the `<Anonymous>` context
itself.

Context checks via `CHECK_CONF` in configuration handlers should check for
this `CONF_ANON` value if the configuration directive should be allowed in the
`<Anonymous>` context.

### The `<Directory>` Context

The `<Directory>` context is for configurations specific to _directories_
(and files), of course.  This includes views of the files in those directories,
based on the logged-in user's name, or group membership, or on the name of the
files (_e.g._ Unix-style "hidden" files), and on whether the user has
permission to see the files.  `.ftpaccess` files occur within this context by
definition; `<Limit>` sections often appear in a `<Directory>` section as well.

Configuration handlers would check for the `CONF_DIR` value in order to make
sure that the configuration directive in question is occurring in an allowed
context.

### The `<Global>` Context

The description in the
[documentation](http://www.proftpd.org/docs/howto/ConfigFile.html#Context) for
the `<Global>` context is good.  Another point to know is that if a directive
is set in this context, and then the same directive is used in the
"server config" or `<VirtualHost>` contexts with different parameters, those
parameters take precedence over the `<Global>` parameters.  This allows you to
configure things for everyone equally, then tweak specific ervers individually,
on a per-server basis.

In configuration directive handlers and context checks, this context uses the
`CONF_GLOBAL` value.

### The `<Limit>` Context

The `<Limit>` context is used to place limits on who and how individual FTP
commands, or groups of FTP commands, may be used.  The configuration context
value for this section is `CONF_LIMIT`.

### The `<VirtualHost>` Context

The [`<VirtualHost>`](http://www.proftpd.org/docs/howto/Vhost.html) section
is used for configuring per-server settings; its value is `CONF_VIRTUAL`.

### The `.ftpaccess` Context

These files are akin to Apache's `.htaccess` files, which are parsed-on-the-fly
configuration files -- with restricted scopes -- that users can place in their
own directories.  _Note_ that `.ftpaccess` are _similar_ to Apache's
`.htaccess` files, they are _not the same_.  For example, ProFTPD's
`.ftpaccess` files do not support a "require" directive, nor Apache's
`AuthRealm` directive.  That particular area of Apache configuration is
targetted for restricting access to anonymous connections; by its nature,
ProFTPD handles anonymous connections as special cases of the normal
authenticated connections.

Configuration directive handlers that wish to allow their configuration
directive to be placed in this context do so via the `CONF_DYNDIR` value.

### Configuration Flags

The `config_rec` structure defines a `flags` member that is used for indicating
different configuration types.  At present, these configuration types include:

* `CF_DEFER`
* `CF_DYNAMIC`
* `CF_MERGEDOWN`
* `CF_MERGEDOWN_MULTI`

The `CF_MERGEDOWN` flag is the most interesting of the flags, and the one the
developer is most likely to encounter.  It is used within a configuration
directive handler like so:

```
  config_rec *c = NULL;

  c = add_config_param(...);
  c->flags |= CF_MERGEDOWN;
```

This flag tells the context-processing functions that configuration
sub-contexts are to _inherit_ this `config_rec` setting, unless the
sub-context already has a `config_rec` of the same directive name.

The other two flags, `CF_DEFER` and `CF_DYNAMIC`, are set automatically for
`<Anonymous>` and `.ftpacces` contexts, respectively.

---

[Table of Contents](../toc.md)

---
