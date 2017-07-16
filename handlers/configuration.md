# ProFTPD Developer Guide: Handlers: Configuration

---

[Table of Contents](../toc.md)

---

## Configuration Handlers

Modules can be used to extend the functionality of the server.  Often that
additional functionality is configurable, and can behave differently according
to the server administrator's needs.  In order to allow the administrator the
ability to tweak the module functionality and behavior, the module often needs
to add its own configuration directives.  Thus, one of the common tasks of the
module developer is the writing of handlers for configuration directives,
checking their format, syntax, and sanity.  The typical configuration directive
handler, depending on its complexity, will allocate any necessary resources and
perform any needed computations on the directive parameters.  Last, a
configuration record containing the processed information will be created, and
stored for later retrieval.

The steps taken during the handling of a configuration directive are
straightforward:

1. Check the number of parameters used in the configuration file
1. Check the configuration contexts in which the directive appeared
1. Parse the given parameters
1. Set a configuration record

### Common Configuration Handler APIs

To help facilitate checking the number of parameters given for the
configuration directive there is the [`CHECK_ARGS`](../api/macros.md#check-args)
macro.

This macro takes the `cmd_rec *` passed to the configuration directive handler,
and the number of parametera that the directive _should_ have.  Note, however,
that the current macro is inadequate, as it only checks that there are _at
least_ the requested number of parameters, but does not produce an error if
there are _more_ than the requested number of parameters.  Other, less
frequently used macros for the checking of number of parameters are
[`CHECK_HASARGS`](../api/macros.md#check-hasargs) and
[`CHECK_VARARGS`](../api/macros.md#check-varargs).

For example, to check that your directive is used with at least two parameters:

```
  CHECK_ARGS(cmd, 2);
```

As with checking the number of parameters, there is a convenience macro
used to check the configuration section (or "context") in which your directive
appeared in the configuration file, against a list of allowed contexts:
[`CHECK_CONF`](../api/macros.md#check-conf).  This macro takes two arguments:
the `cmd_rec *` passed to the configuration directive handler, and a bitset
of context values `OR`'d together; see the "Internals" documentation on
[contexts](../internals/contexts.md).

For example, if your directive should only be used in the `<VirtualHost>`
and `<Directory>` sections:

```
  CHECK_CONF(cmd, CONF_VIRTUAL|CONF_DIR);
```

Now we come to the substance of the configuration directive handler: parsing
the parameters given for proper format, syntax, range, and suitability.  What
the proper format, syntax, range, and suitability _are_ depends on the purpose
of your directive, what kind of parameters it should be accepting, and how
those parameters will be used later by the module code.

First, there is the matter of accessing the parameters that appeared in the
configuration file.  A configuration directive handler accepts a single
argument, a `cmd_rec *`.  The parameter values for the directive are stored
in this structure in a manner similar to what is passed, by the operating
system, to any C program's `main()` function: an `int argc` denoting the
number of parameters, and a `void **argv` containing the array of parameters
as strings.  In the given `cmd_rec *` argument, then, `cmd->argc` is the
number of parameters used **including** the directive name itself, and
`cmd->argv` are those parameters, also including the directive name itself.
The first parameter is the directive name: `cmd->argv[0]`.  The `cmd->argv`
array is `NULL`-terminated; `cmd->argv[cmd->argc]` is always `NULL`.

One common type of configuration directive is a simple on/off Boolean switch.
For parsing a parameter that should be a Boolean value, there is the
`get_boolean()` function.  Many handlers in the core modules' configuration
directive handlers will have code which looks like:

```
  int setting = -1;
  ...
  setting = get_boolean(cmd, 1);
  if (setting == -1) {
    CONF_ERROR(cmd, "expected Boolean value");
  }
```

If the parameter should be a numeric value, make sure that given parameter is
indeed a number (use `strtol(3)` or `strtoul(3)` for their range-checking
abilities as well as string-to-number conversion ability, rather than
`atoi(3)`), and that the number is within the proper range: is it negative or
zero when it should only be positive?  Is it smaller than the largest possible
value in the parameter range?  If the parameter value should be a filesystem
path, check that the given string is an absolute path (if required) and/or
that the path exists, and leads to a file of the correct type (_e.g_
directory, regular file, symbolic link, whatever).  Try to verify and validate
as much as possible at this stage, for it will save you time, debugging, and
error-handling code elsewhere in the module code.

If, during the handling of this directive, a fatal or irrecoverable error is
detected in the directive parameters, use the
[`CONF_ERROR`](../api/macros.md#conf-error) macro to error out of the
configuration handler.

### Configuration Parameter Storage

Once the parameters have been processed, the handler is ready to create and
populate a `config_rec` structure, for storing the parsed data.  The parsing
of the configuration data occurs when the daemon starts up, _and_ when the
daemon is restarted (as when the `SIGHUP` signal is used).  How then to store
configuration parameters in a persistent place, to be looked up later when a
command handler, or an authentication handler, is called?  That persistent
place turns out to be an in-memory bag of configuration records (these
`config_rec` structures), stored in hierarchical _sets_.  This structure
allows for easy retrieval of a record using a string as the lookup key.  So,
the module configuration handlers simply need to know how to add their
configuration data to this structure.

The primary function to use for this is `add_config_param()`.  This function
creates a `config_rec` and stores it in the structure according to the server
and context in which the directive occurred.  This function stores `void`
pointers, so any arbitrary data can be stored, via those pointers, in the
created `config_rec`.  If the parameter data to be stored will be all strings,
as is common, then a similar storage function, `add_config_param_str()` should
be used.  This second function calls `pstrdup()` on each of its arguments,
assuming they are strings. This way you do not have to worry about figuring
out which pool to use for allocating space for those strings.  (Knowing which
pools to use, and their various lifetimes, is discussed
[here](../internals/pools.md).)

In much of the current code, the first argument to either of these functions
is a string literal, usually the name of the configuration directive that
triggered the use of the configuration handler.  However, another way (and
slightly more efficient, as it uses less stack memory) is to use
`cmd->argv[0]` which points to the same string.

### Configuration Directive Flags

One thing to bear in mind when storing configuration parameters is the
contexts in which they will be retrieved.  Is the directive allowed only
in the configuration contexts of "server config", `<VirtualHost>`, or
`<Global>`?  Or will it be useful in the `<Anonymous>` context as well?  Maybe
it pertains to directories, and so will be in `<Directory>` and
`.ftpacces` configuration contexts?  By adding the
[`CF_MERGEDOWN`](../api/macros.md#cf-mergedown) flag to the `config_rec`
returned by `add_config_param()` or `add_config_param_str()`, we can tell the
server that this `config_rec` should be copied and "merged down" into all
lower contexts until it either hits a configuration record with the same name,
or bottoms out.

Here is an example of what the configuration tree looks like, _without_
`CF_MERGEDOWN`, for a fictitious `ExampleDirective`, assuming that directive
appeared within an `<Anonymous>` section in the configuration file:

```
  <VirtualHost>
        |----------\
               <Anonymous>
                   | - ExampleDirective  <------- configuration placed here
                   |-----------\
                           <Directory>   <------- configuration does not apply here...
                               |-----------\
                                        <Limit> <--- or here....
```

If, on the other hand, the `CF_MERGEDOWN` flag is set by the configuration
handler, like so:

```
  config_rec *c = NULL;
  ...
  c = add_config_param("ExampleDirective", 1, (void *) foo);
  c->flags |= CF_MERGEDOWN;
```

then the configuration tree would look like:

```
  <VirtualHost>
        |----------\
               <Anonymous>
                   | - ExampleDirective  <------- configuration placed here
                   |-----------\
                           <Directory>   <------- mergedown means it is **also** placed here
                               | - ExampleDirective
                               |-------------\
                                          <Limit> <-------- and here...
                                             | - ExampleDirective

```

#### Muliple Configuration Records

There is nothing to prevent a given configuration directive from occurring
_multiple times_ in the configuration file.  Indeed, for some configuration
directives this is the desired behavior.  Each time a configuration directive
is encountered, the configuration handler for that directive is called.
Most configuration handlers do not need to do anything special to handle
multiply configured directives.  If that directive be only used once,
however, the configuration handler will need to enforce this.

As most configuration handler functions call `add_config_param()` or
`add_config_param_str()`, a developer might think that a directive that occurs
multiple times might overwrite the previously set `config_rec`.  However,
_each time these functions are called_,  a new `config_rec` is generated
and inserted into a _set_, and that set can handle any number of configuration
records for the same directive.

Finally, if everything is in order, the configuration handler should indicate
to the parsing engine that the directive was successfully processed, using
the [`HANDLED`](../api/macros.md#handled) macro.

### Example Configuration Directive Handler

To demonstrate all of this in action, here is the configuration directive
handler for the [`RootLogin`](http://proftpd.org/docs/modules/mod_auth.html#RootLogin) directive, a configuration directive that accepts only a single Boolean
parameter, as it currently exists in the `mod_core` module:

```
  MODRET set_rootlogin(cmd_rec *cmd) {
    int root_login = -1;
    config_rec *c;

    /* Make sure there's at least one parameter */
    CHECK_ARGS(cmd, 1);

    /* Check that the directive occurs in a legal context */
    CHECK_CONF(cmd, CONF_ROOT|CONF_VIRTUAL|CONF_ANON|CONF_GLOBAL);

    /* Make sure a correct Boolean parameter was given, and return
     * an informative error message if not.
     */
    root_login = get_boolean(cmd, 1);
    if (root_login == -1) {
      CONF_ERROR(cmd, "expected Boolean parameter");
    }

    /* Allocate a config_rec. The NULL is just a temporary placeholder. */
    c = add_config_param(cmd->argv[0], 1, NULL);

    /* Rather than casting the Boolean variable, an int, into the space of
     * a void pointer, allocate the space for the Boolean manually, from
     * the config_rec's own pool.
     */
    c->argv[0] = palloc(c->pool, sizeof(int));

    /* Now store the Boolean parameter. */
    *((int *) c->argv[0]) = root_login;

    /* Since RootLogin is allowed in the <Anonymous> section, we
     * need to set CF_MERGEDOWN.
     */
    c->flags |= CF_MERGEDOWN;

    /* This directive has been successfully handled. */
    return PR_HANDLED(cmd);
  }
```

### Configuration Parameter Retrieval

While discussion of configuration directive lookup/retrieval does not properly
belong in a discussion of configuration handlers, it _does_ follow the
discussion of parameter storage functions.  The way in which these records are
retrieved needs to be handled carefully.  The lookup/retrieval functions are:

* `find_config()`, `find_config_next()`
* `get_param_int()`, `get_param_int_next()`
* `get_param_ptr()`, `get_param_ptr_next()`

So how _does_ the module developer know when to use which lookup API?  It
depends on what kind of information is stored in the record being retrieved.
If that record only holds one parameter, a number or a string or other type
of pointer, then the `get_param_*()` APIs can be used.  The `get_param_int()`
function, despite its name, returns as a `long` value the first parameter in
the matched record (available as `config_rec->argv[0]`).  The
`get_param_int_next()` API can then be used to find and return the value of
the _next_ matching record.  Similarly with the `get_param_ptr()` and
`get_param_ptr_next()` API, excepting that they return `void *`s rather than
`long`s.  The `get_param_ptr()` function is often used to retrieve
configuration data stored as a string.

For more complicated `config_rec`s, such as those that hold multiple pointers
and data such as ACLs, the `find_config()` and `find_config_next()` API is
necessary.  These functions return pointers to the matched configuration
record itself, rather than just the first argument of that record.  Any
`config_rec` stored with more than one parameter will need to be accessed via
these functions.

The next consideration is in which correct configuration _set_ to look for the
desired records.  Which configuration set to use depends on the directive and
the configuration contexts in which it is allowed to appear.  For example,
there are some configuration directives (_e.g._ `DefaultRoot`) that are only
allowed to appear in the configuration sections of "server config",
`<VirtualHost>`, and `<Global>`.  This means that that configuration record
will always be in `main_server->conf`, the top-level set for the server
handling the current session.  If the directive may appear in the
`<Anonymous>` section, use the `CF_MERGEDOWN` flag and the
[`TOPLEVEL_CONF`](../api/macros.md#toplevel-conf) macro as the search set, as
it will expand appropriately to the correct set to use.  The "mergedown" flag,
in this case, is necessary to have a directive that appears in a context
"higher" than `<Anonymous>` appear in that context in the configuration
tree as well.  Confused yet?  For those directives that should cascade down
through every set, use the `CF_MERGEDOWN` flag, and use the
[`CURRENT_CONF`](../api/macros.md#current-conf) macro as the search set.

Failure to use the `CF_MERGEDOWN` flag when necessary, or searching in the
wrong set, leads to "mergedown" bugs, and settings not taking effect when they
should.

#### Example Lookup Code

Here is an example of a lookup loop than can be used to deal with directives
that can be used multiple times:

```
  config_rec *c = NULL;
  ...

  /* See if the DefaultRoot directive has been used */
  c = find_config(main_server->conf, CONF_PARAM, "DefaultRoot", FALSE);
  while (c != NULL) {

    /* Do some processing of the retrieved record here */
    ...

    /* Search for the next occurrence */
    c = find_config_next(c, c->next, CONF_PARAM, "DefaultRoot", FALSE);
  }
```

### Obligations

The last duty of the module developer, and the most important duty (from a
user's point of view), is to **document** what has been developed.  The
existing configuration directive documentation should suffice as a template.
**Please** do everyone a favor and distribute documentation with your module,
so that it can be used without requiring users to browse and decipher the
source code.

---

[Table of Contents](../toc.md)

---

