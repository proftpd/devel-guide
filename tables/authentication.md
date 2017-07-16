# ProFTPD Developer Guide: Tables: Authentication Tables

---

[Table of Contents](../toc.md)

---

## Authentication Tables

The `authtable` associates and authentication type, a name, and the handler
function together:

* authentication type (only type `0` defined at present)
* authentication function name
* authentication handler function

The handler function **must** have the following signature/prototype:

```
  MODRET func(cmd_rec *)
```

Each hander accepts data in from the `cmd_rec` structure, and should
return its results via the `MODRET` structure's `data` field (`MODRET->data`).
The API function `mod_create_data()` can create this data structure properly
for returning to the caller.

### Example Authentication Table

This example authentication table is taken directly from `mod_auth_unix.c`:

```
static authtable auth_unix_authtab[] = {
  { 0, "setpwent", pw_setpwent },
  { 0, "endpwent", pw_endpwent },
  { 0, "setgrent", pw_setgrent },
  { 0, "endgrent", pw_endgrent },
  { 0, "getpwent", pw_getpwent },
  { 0, "getgrent", pw_getgrent },
  { 0, "getpwnam", pw_getpwnam },
  { 0, "getpwuid", pw_getpwuid },
  { 0, "getgrnam", pw_getgrnam },
  { 0, "getgrgid", pw_getgrgid },
  { 0, "auth",     pw_auth },
  { 0, "check",    pw_check },
  { 0, "uid_name", pw_uid_name },
  { 0, "gid_name", pw_gid_name },
  { 0, "name_uid", pw_name_uid },
  { 0, "name_gid", pw_name_gid },
  { 0, NULL }
};
```

The trailing `{ 0, NULL }` _sentinel_ entry marks the end of the authentication
table, and should **always** be used.

---

[Table of Contents](../toc.md)

---
