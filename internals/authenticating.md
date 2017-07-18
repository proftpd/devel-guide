# ProFTPD Developer Guide: Internal: Authenticating Users

---

[Table of Contents](../toc.md)

---

## Authenticating Users

When an FTP client connects to ProFTPD, the client needs to authenticate
itself, as per the FTP protocol.  The client sends a `USER` command followed
by a `PASS` command in most cases.

Upon receiving a `USER` command, ProFTPD will first check to see if the
requested user is allowed to log in, based on any `<Limit LOGIN>` sections.
Then it will check to see if the login is allowed by any configured
`MaxClients`, `MaxClientsPerHost`, and `MaxHostsPerUser` directives.

When ProFTPD receives a `PASS` command, it will again check for configured
`MaxClients`, `MaxClientsPerHost`, and `MaxHostsPerUser` limits.  The first
check, during the handling of the `USER` command, handles `UserAlias`ed
user names; this subsequent check handles _real_ user names.  The actual work
of authenticating the user then takes place (more on this below).  If the
authentication succeeds, any [`DisplayLogin`](http://www.proftpd.org/docs/modules/mod_auth.html#DisplayLogin) or [`AccessGrantMsg`](http://www.proftpd.org/docs/modules/mod_auth.html#AccessGrantMsg) messages are sent to the client.
Otherwise, any [`AccessDenyMsg`](http://www.proftpd.org/docs/modules/mod_auth.html#AccessDenyMsg) message is sent.

During authentication, the process handling the connection is running under
the identity configured using the `User` and `Group` configuration directives.
Only after successfully authenticating the user will the process assume the
identity and privileges of that authenticated user.  This process of
authentication is handled by the `setup_env()` function of the `mod_auth`
module, and is quite involved.

First, this function determines whether the login request is for an
`<Anonymous>` server, or for a real user.  Unknown users are rejected at this
point.  The determination of a "known" user is done via the `pr_auth_getpwnam()`
Auth API.  Modules that wish to supply their own authentication routines will
need to provide [authentication handlers](../handlers/authentication.md).  The
Auth API "cascades" through each of the registered authentication modules of
the server, asking each module in turn to authenticate the user until an
error is encountered, or until a module successfully returns the requested
information.  This design allows multiple authentication schemes to be present
simultaneously.

The `pr_auth_getgroups()` Auth API function is similarly used to request the
supplemental group membership information, both group names and group IDs,
from authentication modules.  The list of obtained group IDs, if any, will
subsequently be passed on to the `setgroups(2)` system call.  At this point, I
would like to point out a common error message pertaining to supplemental
group membership:

```
  "unable to set groups: Invalid argument"
```

This message will occur for one of two reasons: negative IDs for the user,
or the user is a member of more than the _maximum allowable number_ of groups;
see documentation on the `_SC_NGROUPS_MAX` variable of the `sysconf(3)`
library call.

Next, the function will check to see if the requested user is
"root"; logging in as "root" _requires_ that the
[`RootLogin`](http://www.proftpd.org/docs/modules/mod_auth.html#RootLogin)
directive be explicitly set; in general, allowing remote root login a Very
Bad Idea.

At this point, the given password is checked.  If any
[`UserPassword`](http://www.proftpd.org/docs/modules/mod_auth.html#UserPassword)
directives are set, then the configured password will be checked using the
`pr_auth_check()` Auth API function.  Otherwise, the `pr_auth_authenticate()`
Auth API is called.

If the requested user has succeeded up to this point, the default directory in
which to place the user, either their home directory or any applicable
[`DefaultChdir`](http://www.proftpd.org/docs/modules/mod_auth.html#DefaultChdir)
directives, is retrieved.  This path is checked for any applicable `<Limit>`
directives.  Next, after logging a `wtmp` or `utmp` log entry, this function
will check and apply any [`DefaultRoot`](http://www.proftpd.org/docs/modules/mod_auth.html#DefaultRoot)
configuration directives as needed.  Last, the identity of the current process
is changed to be that of the authenticated user.

At this point, unless authentication has failed at any point earlier,
the authentication process is done.  Any [`.ftpaccess`](ftpaccess.md) files
are parsed, the default transfer mode is set (configurable via
[`DefaultTransferMode`](http://www.proftpd.org/docs/modules/mod_xfer.html#DefaultTransferMode)), and the [scoreboard](scoreboard.md) is updated.

Authentication is a complex process, and may fail at several points: if
an authentication module encounters an error while authenticating, if
the login is not for a valid user or is not a valid `UserAlias`, if the login
is denied by an applicable `<Limit LOGIN>`, if the password given was
incorrect or expired, if the account used is disabled, if the user's account
includes an invalid shell, if the login name is listed in the `/etc/ftpusers`
file and the server is configured to honour that file, _etc_.

---

[Table of Contents](../toc.md)

---
