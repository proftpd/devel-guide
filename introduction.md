# ProFTPD Developer Guide: Introduction

---

[Table of Contents](toc.md)

---

ProFTPD is an FTP server modeled around the Apache HTTP server, with a similar
configuration file syntax and modular structure.  In light of this similarity,
I have utilized (_i.e_ _plagiarized_) the
[Apache API](http://httpd.apache.org/docs/misc/API.html) documentation, as many
of the concepts are the same.  Some of the words and explanations below are
thus not mine.

These are some notes on the ProFTPD API and the data structures you have to
deal with, _etc_.  They are not yet nearly complete, but hopefully, they will
help you get your bearings.  Keep in mind that the API is still subject to
change as we gain experience with it.  However, it will be easy to adapt
modules to any changes that are made.

----------
## Handlers, Modules, and Commands

ProFTPD breaks down command handling into a series of simple steps or _phases_,
similar to the way the Apache API breaks down request handling.  These are:

* Preprocess the command
* Process the command
* Postprocess the command
* Log the command

These phases are handled by looking at each of a list of _modules, looking to
see if each of the modules has a handler for the phase, and attempting invoking
that handler if so.  The handler can typically do one of three things:

* _Handle_ the command, and indicate to the processing engine that it should
   consider the command completed, and continue its processing.
* _Decline_ to handle the command, and indicate to the processing engine that
  it should act as if the handler waas never called, and to continue its
  procesing.
* _Signal_ an *error* by returning one of the FTP error codes.  This terminates
  normal handling of the request; the command may be logged.

Most phases are terminated by the first module that handles them.  The handlers
themselves are functions of one argument (a pointer to a `cmd_rec` structure),
which returns a `MODRET` structure.

At this point, we need to explain the structure of a module.  Our candidate
will be one of the simple ones, the "casing" module, which will alter the case
(_e.g._ lowercase or uppercase) of the letters in the name of the file
requested by a client for download, before the server looks up that file.

Let us start with the command handlers.  In order to catch the names of the
requested files before the server retrieves them, the module declares a command
handler that is interested in handling any download commands issued by a client.

This "casing" module will also need code for handling configuration directives,
_e.g._ `DowncaseFileNames` and `UpcaseFileNames`.  To handle these multiple
configuration directives, modules have _tables_ which declare their
configuration directives, and the function which processes/handles the
directive parameters.  The configuration directive handler performs such checks
as whether the configuration directive is in an appropriate section/context,
whether the parameter are correct, _etc_.

A final note on the declared types of the arguments of some of these commands:
a `pool` is a pointer to a _resource pool_ structure; these are used by the
server to keep track of the memory which has been allocated, files opened,
_etc_, either to service a particular command, or to handle the process of
configuration itself.  That way, when the command is over (or, for the
configuration pool, when the server is restarting), the memory can be freed,
and the files closed, _en masse_, without anyone having to write explicit code
to track them down and dispose of them.

## Example Module

With no further ado, the "casing" module itself:

```
  /* Declaration of command handler */
  MODRET fixup_filenames(cmd_rec *cmd);

  /* Declaration of configuration directive handlers */
  MODRET set_lowercase_filenames(cmd_rec *cmd);
  MODRET set_uppercase_filenames(cmd_rec *cmd);

  /* Define the "configuration handler" table, which links configuration file
   * directives with the appropriate handlers in this module
   */
  static conftable casing_conftab[] = {
    { "DowncaseFileNames",  set_downcase_filenames, NULL },
    { "UpcaseFilenames",    set_upcase_filenames,   NULL },
    { NULL }
  };

  /* Define the "command handler" table, which links client-issued commands
   * with the interested handlers in this module
   */
  static cmdtable casing_cmdtab[] = {
    { PRE_CMD, C_RETR, G_NONE, fixup_filenames, TRUE, FALSE },
    { 0, NULL }
  };

  module casing_module = {
    NULL,                   /* Pointer to the next module -- ALWAYS NULL */
    NULL,                   /* Pointer to the previous module -- ALWAYS NULL */
    0x20,                   /* ProFTPD Module API version 2.0 */
    "casing",               /* Module name */
    casing_conftab,         /* Configuration directive handler table */
    casing_cmdtab,          /* Command handler table */
    NULL,                   /* Authentication function table */
    NULL,                   /* Module initialization function */
    NULL                    /* Connection initialization function */
    NULL                    /* Module version */
  };
```

For a real-life example of such a module, see [`mod_case`](https://github.com/Castaglia/proftpd-mod_case).

## Command Handlers

The sole parameter to handlers is a pointer to a `cmd_rec` structure.  This
structure describes a particular command which has been made to the server by
a client.  Each connection by a client generates multiple `cmd_rec` structures,
such as the `USER` FTP command.

The `cmd_rec` contains pointers to a resource pool which will be cleared when
the server is finished handling the command.  The `cmd_rec` also contains
pointers to structures containing per-server information, and most importantly,
information on the command itself.

Also present are pointers to _private data_ that a handler has created, during
the servicing the command (so that modules' handlers for one phase can pass
_notes_ to their handlers for other phases), and to a `server_rec`, which
contains per (virtual) server configuration data.

Most `cmd_rec` structures are built by when the core engine reads and parses
an FTP command from a client, and fills in the fields.  The filled-in `cmd_rec`
is then handed off to command handlers that have registered an interest in
handling that particular FTP command.

## Command Responses

As discussed above, each handler, when invoked to handle a particular
`cmd_rec`, *must* return a `MODRET`; this structure is usually one generated
by one of the provided macros, to indicate what happened.  That can be one of:

* `HANDLED`: The command was handled successfully.  This may or may not
  terminate the phase.
* `DECLINED`: No error condition exists, but the module declined to handle
  this phase of the command.
* `ERROR`: An error has occurred while processing the command, which aborts
  additional handling of the command.

There are two main ways to respond to a client command inside a command handler.
The first, _and preferred_, method of transmitting responses (numeric response
code plus text message) to clients is via the internal response chain.  Using
this allows each handler to add their own individual responses, which will all
then be _flushed_ to the client, after the command successfully completes (or
fails).

The second way is _incompatible with other handlers_, and should **only** be
used if the handler is about to terminate the current connection with the
client.  This second method must be used because when a handler terminates the
client connection itself, the core engine's internal response chain will never
be processed and flushed to the client.

Response chains are covered in more details [here](handlers/responses.md).

## Authentication Handlers

The processing of authentication commands is a little different from the
other FTP commands.

_NOTE: Stuff that should be discussed here_:

* _Authentication commands of `USER`, `PASS` (RFC 2228 `AUTH`, `ADAT`)_
* _`authtab` and specific authentication handlers (`mod_auth_unix` and
  `mod_ldap` examples)_
* relevant FTP error response codes

## Logging Handlers

The logging of commands occurs as part of the handling process, and can be
done at multiple points in the process.

_Stuff that should be discussed here_:

* _`LOG_CMD`, `LOG_CMD_ERR`_
* _`mod_log`, `mod_xfer`, `mod_sample`'s `log_cmd()`_

---
## Resource Allocation and Resource Pools

One of the problems of writing and designing a server is preventing resource
leaking, that is, allocating resources (memory, open files, _etc_), without
subsequently releasing them.  The _resource pool_ machinery is designed to make
it easy to prevent leaks from happening, by allowing resources to be allocated
in such a way that they are _automatically_ released when the server is done
with them.

How does this work?  Memory which is allocated, file opened, _etc_ to deal with
a particular command are tied to a _resource pool_ (or just "pool") which is
allocated for that command.  The pool is a data structure which itself tracks
the resources in question.

When the command has been processed, the pool is _destroyed_.  At that point,
all memory associated with the pool is released/freed, all files associated
with the pool are closed, and any other clean-up functions which are associated
with the pool (for custom releasing _e.g._ external resources) are run.  When
this is over, we can be confident that all the resources tied to that pool have
been released, and that none of them have leaked.

Server restarts, and allocation of memory and resources for per-server
configuration, are handled in a similar way.  There is a _configuration pool_,
which keeps track of resources which were allocated while reading the
configuration files, and handling the directives therein; for instance, the
memory that was allocated for per-server module configuration, log files and
other files that were opened, and so forth.  When the server restarts, and has
to reread the configuration files, the configuration pool is destroyed, and so
the memory and file descriptors which were taken up by reading them the
last time are made available for reuse.

We begin here by describing how memory is allocated to pools, and then
discuss how other resources are tracked by the resource pool
machinery.

### Memory Allocation in Pools

Memory is allocated in pools by calling the function `palloc()`, which takes
two arguments, one being a pointer to a resource pool structure (commonly
seen as `*p`), and the other being the amount of memory to allocate (as an
`int`).  Within command handlers, the most common way of getting a pool
structure is by using the `pool` (or `tmp_pool`, if appropriate) field of the
given `cmd_rec`; hence the repeated appearance of the following idiom in module
code:

```
  MODRET my_handler(cmd_rec *cmd) {
      struct my_structure *foo;
      ...

      foo = palloc(cmd>pool, sizeof(my_structure));
  }
```

Note that _there is no `pfree()` function_; `palloc()`ed memory is freed only
when its `pool` is destroyed.  This means that `palloc()` does not have as much
accounting as `malloc(3)`; all it does, in the typical case, is to round up the
size, bump a pointer, and do a range check.

### Allocating Initialized Memory

There are functions which allocate initialized memory, which is frequently
useful.  The function `pcalloc()` has the same interface as `palloc()`, but
zeros out the memory allocated before returning it.  The function `pstrdup()`
takes a pool and a `const char *` as arguments (`pstrndup()` takes an
additional `size_t` length), allocates memory for a copy of that string,
and returns a pointer to the copy.  Finally, `pstrcat()` is a varargs-style
function, which takes a pointer to a pool, and the additional arguments, of
which the _last_ one *must* be `NULL`. The function allocates enough contiguous
memory to fit copies of each of the strings, as a unit; for example:

```
  pstrcat(cmd->pool, "foo", "/", "bar", NULL);
```

returns a pointer to 8 bytes worth of memory, initialized to `"foo/bar"`.

For almost everything folks do, `cmd->pool` is the pool to use.  For memory
needed _just for the duration of the handler function_, `cmd->tmp_pool` is
more appropriate.  This "temporary" pool is destroyed after each handler is
called, and a new pool created before calling the next handler, for the same
`cmd_rec`.

### Additional Pool Cleanups

Pool _cleanups_ live until `destroy_pool()` is called; all associated cleanups
on the pool will be invoked, and then the memory for the pool will be freed.
Cleanup functions are _callbacks_ that are explicitly registered with a pool
using `register_cleanup()`.

## Configuration Directives

Most modules require/provide some sort of configuration, in the form of
configuration directives.  So how does a module handle such directives?
In our "casing" module example, handling directives involves processing the
actual `DowncaseFileNames` and `UpcaseFileNames` directives.  A module
declares all of the configuration directives it wants to handles/process via
its _configuration table_.  The core processing engine, given a parsed
directive, will then see which modules are interested in that directive; this
process is very similar to how commands are processed.  The _configuration
table_ contains information on what directives the module handles and the
associated configuration handler function.  Without further ado, let us look at
the `DowncaseFileNames` configuration handler, which looks like this:

```
  MODRET set_lowercase_filenames(cmd_rec *cmd) {
    int use_lowercase = -1;
    config_rec *c = NULL;

    /* Make sure the directive was given one, and only one, parameter. */
    CHECK_ARGS(cmd, 1);

    /* Check the context in which the directive was used, making sure that
     * it was one of the allowed contexts of "server config", <Anonymous>,
     * <Limit>, or <VirtualHost>.
     */
    CHECK_CONF(cmd, CONF_ROOT|CONF_ANON|CONF_LIMIT|CONF_VIRTUAL);

    /* Use get_boolean() to parse the _first_ parameter as Boolean value. */
    use_lowercase = get_boolean(cmd, 1);
    if (use_lowercase == -1) {
      CONF_ERROR(cmd, "requires a Boolean value");
    }

    c = add_config_param("DowncaseFileNames", 1, NULL);
    c->argv[0] = palloc(c->pool, sizeof(int));
    *((int *) c->argv[0]) = use_lowercase;

    /* Merge this configuration directive "down", so that it affects any
     * contained contexts.
     */
    c->flags |= CF_MERGEDOWN;

    return PR_HANDLED(cmd);
  }
```

This is a fairly typical configuration handler.  As you can see, it takes only
one argument, a `cmd_rec` pointer.  That structure contains a bunch of fields
which are frequently of use to some, but not all, directives, including a pool
(from which memory can be allocated, and to which cleanups should be tied), and
the (virtual) server being configured, from which the module's per-server
configuration data can be obtained if required.

It is also fairly typical in its checking of the configuration directive
parameters.  The number of parameters is checked with `CHECK_ARGS`, which in
this case requires that only one parameter be used with the directive.  Next,
the configuration context is checked with `CHECK_CONF`.  Finally, since this
configuration directive needs only a _true_ or _false_ value, the given
parameter is parsed as a Boolean value, and an error generated if this is not
the case.

The `DowncaseFileNames` configuration directive will automatically be stored in
the in-memory configuration database for the containing server, either "server
config" (for anything outside of `<Anonymous>`, and `<VirtualHost>` contexts),
`<Anonymous>`, or `<VirtualHost>`.  The `server_rec` for the containing server
of the configuration directive's `cmd_rec` is pointed to by `cmd->server`.

The "casing" module configuration table has entries for these directives,
which look like this (as seen above):

```
  static conftable casing_conftab[] = {
    { "DowncaseFileNames",  set_downcase_filenames, NULL },
    { "UpcaseFilenames",    set_upcase_filenames,   NULL },
    { NULL }
  };
```

The entries in these tables are:

* The name of the configuration directive
* The function which handles the directive
* A pointer which is set to the owning module when the module code is
  compiled; should always be set to `NULL` in the table

Finally, having set this all up, we have to use it.  This is ultimately done
in the module's handlers, specifically for its filename handler, which looks
more or less like this:

```
  MODRET fixup_filenames(cmd_rec *cmd) {
    char *new_filename = NULL;
    config_rec *downcase = NULL, *upcase = NULL;

    /* Check the current configuration context for the configuration
     * directive Boolean value, true or false.  If false, return now.
     */
    downcase = find_config(CURRENT_CONF, CONF_PARAM, "DowncaseFileNames", FALSE);
    upcase = find_config(CURRENT_CONF, CONF_PARAM, "UpcaseFileNames", FALSE);

    if (downcase == NULL &&
        upcase == NULL) {
      /* No module directives used; no adjusting required. */
      return PR_DECLINED(cmd);
    }

    /* Get an adjusted requested filename. */
    if (downcase != NULL) {
      int use_lowercase;

      use_lowercase = *((int *) downcase->argv[0]);
      if (use_lowercase == TRUE) {
        new_filename = adjust_filename(cmd->server->pool, cmd->arg, PR_CASE_DOWN);
      }

    } else if (upcase != NULL) {
      int use_uppercase;

      use_uppercase = *((int *) upcase->argv[0]);
      if (use_uppercase == TRUE) {
        new_filename = adjust_filename(cmd->server->pool, cmd->arg, PR_CASE_UP);
      }
    }

    if (new_filename != NULL) {
      /* Copy the new filename into the cmd_rec, for use by the remaining handlers. */
      sstrcpy(cmd->arg, new_filename, strlen(new_filename));
    }

    /* Done with adjustments; let the remaining handlers continue processing. */
    return PR_DECLINED(cmd);
  }
```

The `DowncaseFileNames` or `UpcaseFileNames` configuration directives are
retrieved from the in-memory database using `find_config()`.  If neither
directive applies to the file requested for the `RETR` command, the handler
returns `PR_DECLINED`, and processing continues on to the next handler that is
registered for this command.  On the other hand, if one of the configuration
directives _does_ apply, the "fixup" is done on the filename, then the handler
returns with `PR_DECLINE`, letting other handlers work on the `cmd_rec`,
which now has the adjusted filename.

The registration of the above function as a command handler for downloads
(_i.e._ for the FTP `RETR` command) is done in the _command table_, shown
earlier:

```
  { PRE_CMD, C_RETR, G_NONE, fixup_filenames, TRUE, FALSE },
```

The writing of the `adjust_filename()` function is left as an exercise to
**you**, the budding module developer.

---

[Table of Contents](toc.md)

---
