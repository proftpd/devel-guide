# ProFTPD Developer Guide: Module Development: SITE Commands

---

[Table of Contents](../toc.md)

---

## Custom `SITE` Commands

Adding your own custom `SITE` is straightforward.  You need to define a
`SITE` command handler in your module.

There should be two parts to a custom `SITE` command handler:

* A part that handles the module-specific `SITE` command
* A part to handle adding a _description_ of that command to the `SITE HELP`
  response

### Example `SITE` Command

The example command handler below implements `SITE TIME`:

```
MODRET site_time(cmd_rec *cmd) {

  /* Make sure it's a valid SITE TIME command, with no additional
   * parameters sent by the client.
   */
  if (cmd->argc != 2) {
    return PR_DECLINED(cmd);
  }

  /* This case does the work of returning the local time to the client. */
  if (strcasecmp(cmd->argv[1], "TIME") == 0) {
    char timestr[80] = {'\0'};
    time_t now = time(NULL);
    struct tm *tnow = NULL;

    /* If the user is required to be authenticated/logged in, do the
     * necessary check.  This may or may not be required, depending on
     * the SITE command being implemented.
     */
    if (get_param_int(cmd->server->conf, "authenticated", FALSE) != TRUE) {
      pr_response_send(R_530, "Please login with USER and PASS.");
      return PR_ERROR(cmd);
    }

    /* Check for <Limit> restrictions */
    if (!dir_check(cmd->tmp_pool, "SITE_TIME", "MISC", session.cwd, NULL)) {
      pr_response_add_err(R_550, "%s: %s", cmd->arg, strerror(EPERM));
      return PR_ERROR(cmd);
    }

    /* Log that the user requested the time. */
    pr_log_pri(PR_LOG_INFO, "SITE TIME requested by user %s", session.user);

    /* Prepare the time string to be returned. */
    tnow = localtime(&now);
    if (tnow == NULL) {
      /* If, for some reason, an error occurs, use the 202 error code.
       * Other legal error codes for a SITE command include 500 (syntax
       * error, command not recognized), 501 (syntax error in parameters or
       * arguments), and 530 (not logged in).
       */
      pr_response_add(R_202, "Unable to provide the time right now");

      /* Of course, if something like localtime(3) fails, it should be
       * logged as well.
       */
      pr_log_pri(PR_LOG_NOTICE, "notice: localtime() returned NULL?!?");

      return PR_HANDLED(cmd);
    }

    /* Print the local time into the string buffer. */
    strftime(timestr, sizeof(timestr), "%a %b %d %T %Z %Y", tnow); 
    timestr[sizeof(timestr)-1] = '\0';

    /* The 200 response code is the normal value for such "generic" commands
     * as site-specific SITE commands.
     */
    pr_response_add(R_200, "The current time is: %s", timestr);

    /* If you want to add a multiline response, use the R_DUP macro, like
     * so:
     *
     *  pr_response_add(R_DUP, ...);
     */

    return PR_HANDLED(cmd);
  }

  if (strcasecmp(cmd->argv[1], "HELP") == 0) {
    /* Add a description of SITE TIME to the output.  The response code
     * 214 is the appropriate value to use for "helpful" responses.
     */
    pr_response_add(R_214, "TIME");

    /* If this SITE command has needed a parameter (RFC 959 only allows
     * one parameter to the SITE command, although not many servers seem to
     * strictly enforce this), this HELP line would have looked something like:
     *
     *  pr_response_add(R_214, "TIME <param>");
     */
  }

  /* This defaults to returning DECLINED in the case of a SITE HELP command.
   * This may seem wrong, but is actually correct: the case above adds
   * a response to the response chain for the current command, but does
   * not actually handle the command.  By returning DECLINED now, it
   * allows other modules to add their own SITE HELP responses to the chain
   * before the full chain is flushed back to the client once the command
   * is "officially" handled by mod_site.
   */

  return PR_DECLINED(cmd);
}
```

The only remaining task is to register this command handler by adding
an appropriate entry to the module `cmdtable`:

```
  { CMD, C_SITE, G_NONE, site_time, FALSE, FALSE, CL_MISC },
```

Simple.  The key points to keep in mind are:

* Use the correct response codes as appropriate
* Provide support for the `SITE HELP` command _as well as_ for the custom
  functionality
* Require authentication as necessary
* Perform as many checks on the input as necessary
* Document any custom `SITE` commands supported by your module

Note that the `requires_auth` field of the above `cmdtable` entry (_i.e._ the
first `FALSE`) is `FALSE`, and that the example handler performs the same
authentication check (requiring that the user be authenticated before
using the command) as if `requires_auth` had been `TRUE`.
_This is normal_, and in general a good idea: let your `SITE` handlers
determine themselves if and how to perform authentication/authorization checks,
rather than always relying on the core engine checking.  Your `SITE` handlers
might have different needs/criteria for their usage.

The `CL_MISC` value indicates that the registered `SITE` command is
"miscellaneous" for purposes of logging by `ExtendedLog`.  Custom `SITE`
commands should always use this logging class so that administrators'
`ExtendedLog` files properly show when that particular `SITE` command is used.

---

[Table of Contents](../toc.md)

---
