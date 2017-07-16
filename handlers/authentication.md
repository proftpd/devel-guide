# ProFTPD Developer Guide: Handlers: Authentication

---

[Table of Contents](../toc.md)

---

## Authentication Handlers

Strictly speaking, the authentication handlers are used for both
_authentication_ (verifying that a user is who she claims to be) and
_authorization_ (determining what a user can and cannot do).  A password is
often used for authentication; a user's UID and GID provide "privileges" that
are used for authorization.  This combination of functionality separates
ProFTPD "auth" modules from PAM modules; PAM modules are used for
authentication only.

Authentication handlers allow a module to provide account information
(_e.g._ password, IDs, home directory, and shell) and UID/GID-to-name and
name-to-UID/GID mappings (used for directory listings).  In order for a module
to provide this functionality, it must provide an "authentication table", and
use the handler _names_ listed below.

Each handler accepts data via the `cmd_rec` structure and should return its
data via the `MODRET->data` member.  The `mod_create_data()` function is
usually used by an authentication handler to create and populate this data
structure properly, for returning to the caller.

### `setpwent`

Provides a custom implementation of the `setpwent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_setpwent()` function.

### `setgrent`

Provides a custom implementation of the `setgrent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_setgrent()` function.

### `endpwent`

Provides a custom implementation of the `endpwent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_endpwent()` function.

### `endgrent`

Provides a custom implementation of the `endgrent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_endgrent()` function.

### `getpwent`

Provides a custom implementation of the `getpwent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getpwent()` function.

### `getgrent`

Provides a custom implementation of the `getgrent(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getgrent()` function.

### `getgroups`

Provides a list of supplemental groups for the given user, providing a custom
implementation of the `getgroups(2)` system call.  The handler is accessed via
the Auth API `pr_auth_getgroups()` function.

### `getpwnam`

Provides a custom implementation of the `getpwnam(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getpwnam()` function.

### `getgrnam`

Provides a custom implementation of the `getgrnam(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getgrnam()` function.

### `getpwuid`

Provides a custom implementation of the `getpwuid(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getpwuid()` function.

### `getgrgid`

Provides a custom implementation of the `getgrgid(3)` library function.  The
handler is accessed via the Auth API `pr_auth_getgrgid()` function.

### `auth`

Provides a function to authenticate a user, in whatever manner the developer
wishes.  The user name and the cleartext password, from the _e.g._ `USER` and
`PASS` FTP commands, are passed via `cmd_rec`:

```
    cmd->argv[0] = <i>user-name</i>
    cmd->argv[1] = <i>cleartext-password</i>
```

The handler is accessed via the Auth API `pr_auth_authenticate()` function.
**Note** that it is **critical** that an `auth` handler return one of the
following codes, in case of failed authentication, to indicate the reason:

* `PR_AUTH_AGEPWD`
* `PR_AUTH_BADPWD`
* `PR_AUTH_DISABLEDPWD`
* `PR_AUTH_NOPWD`
* `PR_AUTH_ERROR`

Failure to do so will lead to misleading error responses/messages being sent
to the client.

### `check`

Provides a function for comparing the supplied password against a stored
password _hash_, in whatever manner the developer wishes.  The user name, the
given cleartext password, and the retrieved password hash are passed via
`cmd_rec`:

```
  const char *password_hash = cmd->argv[0];
  const char *user_name = cmd->argv[1];
  const char *cleartext_password = cmd->argv[2];
```

The handler is accessed via the Auth API `pr_auth_check()` function.

### `uid2name`

Used to map a UID to a user name.  The UID to be mapped is passed via
`cmd_rec`:

```
  uid_t uid = *((uid_t *) cmd->argv[0]);
```

The handler is accessed via the Auth API `pr_auth_uid2name()` function.

### `gid2name`

Used to map a GID to a group name.  The GID to be mapped is passed via
`cmd_rec`:

```
  gid_t gid = *((gid_t *) cmd->argv[0]);
```

The handler is accessed via the Auth API `pr_auth_gid2name()` function.

### `name2uid`

Used to map a user name to a UID.  The name to be mapped is passed via
`cmd_rec`:

```
  char *user_name = cmd->argv[0];
```

The handler is accessed via the Auth API `pr_auth_name2uid()` function.

### `name2gid`

Used to map a group name to a GID.  The name to be mapped is passed via
`cmd_rec`:

```
  char *group_name = cmd->argv[0];
```

The handler is accessed via the Auth API `pr_auth_name2gid()` function.

## Using the Auth API

An authentication module's handlers are used at various times during the
lifetime of the server:

* At server startup, `getpwnam` and `getgrnam` handlers are called to resolve
  the [`User`](http://proftpd.org/docs/modules/mod_core.html#User) and
  [`Group`](http://proftpd.org/docs/modules/mod_core.html#Group) configuration
  directives.
* `getpwnam` handlers are used to obtain account information for the name
  for the `USER` FTP command when a client is logging in.
* `getgrent` and/or `getgroups` handlers are used to obtain a list of
  supplemental groups for the authenticating user.
* `auth` handlers are invoked to authenticate a username and cleartext
  password, after a `PASS` FTP command is received from the client.  A
  similar function to this is the `check` handler, whose purpose is to
  support configuration directives such as [`UserPassword`](http://proftpd.org/docs/modules/mod_auth.html#UserPassword).
* `uid2name` and `gid2name` handlers are used to resolve IDs to names for
  legible directory listings.

### Authentication Module Order

The "cascading" order of authentication handlers is similar to that of command
handlers:

1. If an authentication handler returns `DECLINED`, handlers in other modules
   are called in module load order, just as with command handlers.  If _all_
   handlers for a particular function return `DECLINED`, the core engine will
   assume that the operation **failed**.

1. If an authentication handler returns `HANDLED`, the core engine assumes that
   the operation completed successfully, and will stop calling module
   authentication handlers.  Most authentication handlers _must_ return data
   if they complete successfully.  Note that some `void`-style handlers do not
   have this requirement, _e.g._ `setpwent` and friends.

1. If an authentication handler returns `ERROR`, the core engine assumes that
   an error has occured, and stops calling other handlers.  Some authentication
   handlers, `auth`, for example, should use the `ERROR_INT` macro to return
   a numeric error code, such as one of the `PR_AUTH_*` macros in
   [include/auth.h](https://github.com/proftpd/proftpd/blob/master/include/auth.h),
   indicating the reason for failure.

In order for this "cascading" mechanism to function properly and as expected,
the authentication module author _should_ provide handlers for all of the
authentication handlers.  Failure to do so means that the authentication module
provides an _incomplete_ authentication layer, and cascaded calls will appear
to "skip" the module unexpectedly.  At present, the `mod_auth_pam` module
exhibits this behavior: it is an authentication module, but provides no
mechanisms for providing group information or IDs.  It solely provides an
`auth` handler for a "yes or no" answer.

Note that the [`AuthOrder`](http://proftpd.org/docs/modules/mod_core.html#AuthOrder)
directive lets the administrator _explicitly_ configure the order in which
authentication modules are called, instead of implicitly relying on module
load order.

---

[Table of Contents](../toc.md)

---

