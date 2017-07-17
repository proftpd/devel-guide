# ProFTPD Developer Guide: Module Development: Session Credentials

---

[Table of Contents](../toc.md)

---

## Session Credentials

The credentials of a session, or its _identity_, are available via members of
the global `session` variable.  The credentials include user information,
group membership information, and `Class` information (if used).

### Session User

These members of `session` are all set _after_ the `PASS` command has been
_successfully_ processed:

* `session.uid`: User UID, as supplied by the Auth API.
* `session.user`: During authentication, this member is set to the
  _authenticated_ user name.  Note that this user name may be different from
  the user name originally sent by the client (via the `USER` command) if
  [`UserAlias`](http://www.proftpd.org/docs/modules/mod_auth.html#UserAlias)
  has been used.  If authentication fails for any reason, this is set to `NULL`.
* `session.ident_user`: Name of the remote user as established using the ident
   protocol ([RFC 1413](https://www.ietf.org/rfc/rfc1413.txt)), once the
   connection to the server has been made by the client, before any FTP
   commands are processed.  If `mod_ident` is not present, _or_ if
   [`IdentLookups`](http://www.proftpd.org/docs/modules/mod_ident.html#IdentLookups) is set to _off_, this will be the value "UKNOWN".
* `session.anon_user`: Only has a value if the login is an anonymous one;
   otherwise, its value will be `NULL`.  For anonymous logins, this member
   will contain the "password" (usually an email address) sent by the client.
   However, if [`AnonRequirePassword`](http://www.proftpd.org/docs/modules/mod_auth.html#AnonRequirePassword) is set to _on_, the value will be the original
   `USER` sent by client.

In addition to the above `session` members, there is an additional place in
which user identity information, specifically, the user name originally sent
by the client in the `USER` command, is stored: in the session _notes_, and is
retrievable like this:

```
  const char *user;

  user = pr_table_get(session.notes, "mod_auth.orig-user", NULL);
  if (user != NULL) {
    ...
  }
```

### Session Groups

These members are all set after the `PASS` command has been _successfully_
processed:

* `session.gid`: GID of the user's primary group, as supplied by the Auth API.
* `session.group`: Name of the user's primary group, as supplied by the Auth
  API.
* `session.gids`: List (`array_header`) of the GIDs of the supplemental groups
  to which the user belongs.
* `session.groups`: List (`array_header`) of the names of the supplemental
  groups to which the user belongs.

### Session Class

Once a client connects, the class of that client is determined.  The value
is stored in `session`:

* `session.class`: If classes have been configured, this member will contain
   a pointer to the `class_t` struct for the current session.  It is set
   once the client has connected to the server, _before_ any modules'
   session initialization handlers have been run, and before any FTP commands
   are processed.

---

[Table of Contents](../toc.md)

---
