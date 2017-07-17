# ProFTPD Developer Guide: Internals: Process Lifecycle

---

[Table of Contents](../toc.md)

---

## Process Lifecycle

The aim here is to provide a walkthrough of an FTP process, from start to
finish, in detail.  This will help to know what the server is doing at a
given point during a connection, and at what points modules are called upon to
do their magic during the handling of that connection.

The basic model is a core engine which provides services via APIs to pluggable
modules that do all the work.  The core engine handles the difficult work such
as filesystem I/O, network I/O, timers, and calling out to modules, at various
points during the process lifecycle.

### Server Startup

First, the server must be started.  On startup, the server performs the
following tasks, in order:

* Parse and handle command-line parameters
* Initialize subsystems
* Call all module initialization functions
* Parse the configuration files

At this point, the behavior of the server process depends on the
[`ServerType`](http://www.proftpd.org/docs/howto/ServerType.html) configuration
directive.  If set to `inetd`, the process will immediately start processing
the given file descriptor as a client connection.  If instead `standalone` is
configured, the process will daemonize, separating itself from the parent
process, open the necessary sockets, and start listening for connections.

The server process calls `fork_server()` to handle a connecting client.  Again
the `ServerType` directive changes the behavior of the process; when acting as
a `standalone` server, the process _forks a new process_ to handle the client.
**This is an important point to remember**.  Changes made to the child process
(also called the _session process_) will **not** be visible to the parent
daemon process.

For `inetd` servers, the `inetd/xinetd` daemon is the parent process.  From
this point on, the process handles the connection as an FTP session.

### Connection Initialization

Before handling any FTP commands the client may have sent, the session
process has some preparations to do:

* Close any listening sockets; the child process no longer needs those file
  descriptors
* Register signal handlers for `SIGUSR1` and `SIGUSR2`
* Prepare for logging, either to `syslog` or to the `SystemLog`/`ServerLog`
  files
* Set protocol options on the TCP connection
* Lookup the vhost configuration to be used for handling this client
* Perform (optionally) reverse and reverse-reverse DNS resolution
* Check for any scheduled shutdowns in the `/etc/shutmsg` file
* Check for any applicable `<Limit LOGIN>` sections
* Lookup the appropriate
  [`<Class>`](http://www.proftpd.org/docs/howto/Classes.html) for the
  connected client, if any
* Call all module session initialization functions
* Set the session resource limits

### Handling Clients

Now the session process enters `cmd_loop()`, an endless loop for processing
FTP commands and issuing responses.  Before the process handles the first
command, though, it displays the server welcome message to the client.
Commands are handled via command handlers, a separate cycle described
[here](../handlers/commamd.md).

All commands are checked against `Allow` and `Deny` filters.  In addition,
some commands are not allowed prior to authentication; this access check is
performed during the command dispatch cycle as well.  Allowed commands are
recorded in the scoreboard entry for the session, and then dispatched to the
modules.

Once a session has ended, either because the client gave the `QUIT` command,
the TCP connection closed, the session timed out, or some more sinister
mistake occurred, the `pr_session_end()` function is called to clean up after
the client.

---

[Table of Contents](../toc.md)

---
