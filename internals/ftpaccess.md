# ProFTPD Developer Guide: Internals: `.ftpaccess` Files

---

[Table of Contents](../toc.md)

---

## `.ftpaccess` Files

An `.ftpaccess` file is meant to function like Apache's `.htaccess` file: a
file that acts as free-floating section of the server's configuration file.
If an `.ftpaccess` file is present in a directory in which the server looks,
it will parse that `.ftpaccess` file as a configuration file, and act
accordingly.

Only some configuration directives are allowed in the `.ftpaccess` context,
though. The advantage of having this capability is that users can customize
how the server treats directories that are under the user's control via files
placed in those directories, instead of allowing the user to modify the main
server configuration file itself.  The disadvantage is that a user is capable
of possibly overriding a configuration value that was set in the main
configuration file for a specific purpose.

The server treats a directory that contains an `.ftpaccess` file exactly as if
the configuration directives in that file had been placed in a `<Directory>`
section in the main `proftpd.conf` file.  For example, if there is a
`/home/users/bob` directory on your system, and in that directory there was an
`.ftpaccess` file that contained:

```
  DirFakeUser on ~
  DirFakeGroup on ~
  Umask 0077
```

it would be treated exactly as if the following was in the `proftpd.conf` file:

```
  <Directory /home/users/bob>
    DirFakeUser on ~
    DirFakeGroup on ~
    Umask 0077
  </Directory>
```

---

[Table of Contents](../toc.md)

---
