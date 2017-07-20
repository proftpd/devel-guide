# ProFTPD Developer Guide: Internals: Scoreboard

---

[Table of Contents](../toc.md)

---

## Scoreboard

The [Scoreboard](http://www.proftpd.org/docs/howto/Scoreboard.html) is a file
used by the daemon to store statistics about each current session.  It records
things such as the process ID of the session, the IP address of the client and
server involved, the last command issued, the current working directory, file
transfer status, _etc_.  All of this information is stored in a single file,
located using the[`ScoreboardFile`](http://www.proftpd.org/docs/modules/mod_core.html#ScoreboardFile) directive.
This file uses a fixed-length record binary format.  This means that the file
is *not* easily legible; however, the binary format makes for faster I/O.
This is important, as the daemon is _constantly_ reading and writing to the
scoreboard, updating fields for every current session.

### How the Scoreboard is Used

The scoreboard is used by the `ftpcount`, `ftptop`, and `ftpwho` utilities,
for reporting on the status of the running ProFTPD daemon and its current
sessions.  The daemon itself makes use of the scoreboard by using it to track
and enforce such limits as `MaxClients`, `MaxClientsPerHost`,
`MaxClientsPerUser`, and `MaxHostsPerUser`.

### Scoreboard API

The primary object of interest, when dealing with the scoreboard
programmatically, is the `pr_scoreboard_entry_t` structure, shown below:

```
  typedef struct {
    pid_t sce_pid;
    uid_t sce_uid;
    gid_t sce_gid;
    char sce_user[32];
    p_in_addr_t *sce_server_ip;
    int sce_server_port;
    char sce_server_addr[80], sce_server_name[32];
    char sce_client_addr[80];
    char sce_class[32];
    char sce_cwd[PR_TUNABLE_SCOREBOARD_BUFFER_SIZE];
    char sce_cmd[PR_TUNABLE_SCOREBOARD_BUFFER_SIZE];
    time_t sce_begin_idle, sce_begin_session;
    off_t sce_xfer_size, sce_xfer_done;

  } pr_scoreboard_entry_t;
```

For code that wishes to deal with the scoreboard, the Scoreboard API
functions are as follows (from [include/scoreboard.h](https://github.com/proftpd/proftpd/blob/master/include/scoreboard.h)):

```
  const char *pr_get_scoreboard(void);
  int pr_set_scoreboard(const char *file);
  int pr_close_scoreboard(void);
  void pr_delete_scoreboard(void);
  int pr_open_scoreboard(int mode, pid_t *daemon_pid);
  int pr_restore_scoreboard(void);
  int pr_rewind_scoreboard(void);
  int pr_scoreboard_entry_add(void);
  int pr_scoreboard_entry_del(unsigned char);
  pr_scoreboard_entry_t *pr_scoreboard_entry_read(void);
  int pr_scoreboard_entry_update(pid_t _pid_, ...);
```

The `pr_get_scoreboard()` function simply returns the full path to the current
scoreboard file.

The `pr_set_scoreboard()` function is used to change the location of the
scoreboard file.  This should be called before the scoreboard is opened.

The `pr_close_scoreboard()` function does just that, closing the scoreboard
file and releasing any locks that may be present on the scoreboard.

The `pr_delete_scoreboard()` function is used as part of the daemon cleanup,
during its shutdown process, to remove the scoreboard.  There is no reason to
leave a scoreboard around when the daemon is not running (unless the daemon
has a `ServerType` of "inetd").

`pr_open_scoreboard()` is called to open the scoreboard.  The given _mode_ is
the same as that given to `open(2)`, and is usually `O_RDONLY`.  If non-`NULL`,
_daemon_pid_ will point to the process ID of the currently running daemon for
the scoreboard upon return of the function.

The `pr_restore_scoreboard()` is used when done reading the scoreboard.  It
restores the file position pointer of the scoreboard to the position it had
before `pr_rewind_scoreboard()` was called.  Failure to call this function
after using `pr_rewind_scoreboard()` can result in corruption of the scoreboard.

The `pr_rewind_scoreboard()` is used only when the scoreboard is to be read
through in its entirety.  Prior to interating through the scoreboard entries,
this function is called to position the file position pointer at the start of
the scoreboard.  Once done iterating, call `pr_restore_scoreboard()` to
restore the file position pointer to its previous location, prior to the scan
of the scoreboard.

The `pr_scoreboard_entry_add()` function is called when a client connects to
the daemon, allocating a slot in the scoreboard to store the statistics for
the session.

The `pr_scoreboard_entry_del()` function is called when the session is ending,
to remove the scoreboard entry allocated for the session from the scoreboard
file.

The `pr_scoreboard_entry_read()` function reads the _next_ scoreboard entry
from the scoreboard, and returns a pointer to that entry; internally that
entry is stored in a static buffer.

Finally, the `pr_scoreboard_entry_update()` function is used to update each of
the various fields of a scoreboard entry, each at different points during the
lifetime of the session.

These functions, however, are for use by modules, and daemon code.  For
external utilities, _e.g._ `ftptop` and `ftpwho`, there is a slightly
different API (see
[utils/utils.h](https://github.com/proftpd/proftpd/blob/master/utils/utils.h)):

```
  const char *util_get_scoreboard(void);
  int util_set_scoreboard(const char *file);
  int util_close_scoreboard(void);
  int util_open_scoreboard(int mode, pid_t *daemon_pid);
  pr_scoreboard_entry_t *util_scoreboard_read_entry(void);

```

These are functionally the same as their `pr_`-prefixed equivalents, except
that they do not depend on the core code, thus avoiding many dependencies
irrelevent to an external utility.  These routines are read-only, since there
is no valid reason for an external utility to be changing the contents of the
scoreboard; such changes are for the daemon to make.

---

[Table of Contents](../toc.md)

---
