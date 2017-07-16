# ProFTPD Developer Guide: Handlers

---

[Table of Contents](../toc.md)

---

## Handlers

There are three main types of handler functions for ProFTPD: configuration
directive handlers, command handlers, and authentication handlers.

### Configuration Directive Handlers

[Configuration directive](configuration.md) handlers are used to process the
configuration directives encountered in the main `proftpd.conf` configuration
file, any `Include`d configuration files, and any `.ftpaccess` files
encountered.  The context in which these functions operate is one _without_
connected clients; much of the "processing" entails syntax and availability
checks of directive parameters, and the recording of the configured settings.

### Command Handlers

The [command](command.md) handlers are responsible for the processing of the
client-sent FTP commands; these are the heart of the server, providing the
services requested by the connected user.

### Authentication Handlers

The last type, [authentication](authentication.md) handlers, are not really
handlers so much as they are replacement functions, to be used by ProFTPD for
the purposes of authentication (and authorization) _in place of_ the system or
library-provided functions.  These types of handlers are not as common as the
other two types of handlers.  For the curious reader, however, the
sources for the `mod_ldap`, `mod_auth_pam`, and `mod_auth_unix` modules
illustrate how these handlers are implemented.  Note that the tracing of the
calling of these functions during a live connection can be tricky.

### Response Handlers

While not quite true "handlers", _per se_, the [response](responses.md)
functions are also important piece of module handler development.  These
functions deal with how ProFTPD sends its responses back to the connected
clients.

### Handler Macros

There are also several pre-defined macros to help in the checking of various
handler argument conditions, such as argument number and syntax.  These macros
are described [here](macros.md).

---

[Table of Contents](../toc.md)

---
