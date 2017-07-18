# ProFTPD Developer Guide: Internals: Configuration Expressions

---

[Table of Contents](../toc.md)

---

## Configuration Expressions

There are many configuration directives that can take, as a parameter, a list
of names, be they user, group, or class names.  These lists are then evaluated
as Boolean `AND` or `OR` expressions.  Note that such name lists are very
different from _regular_ expressions, which are used for pattern matching.

### `AND` Expressions

An `AND` expression is simply one in which _all_ of the elements in the list
must evaluate to `TRUE` for the entire expression to be `TRUE`.  Such an
expression is usually useful when used on group names, for a user may be a
member of multiple groups.  For example, given a group expression like:

```
  group1,group2
```

if evaluated as an `AND` expression, it will evaluate to `TRUE` if the current
user is both a member of "group1" **and** "group2".  Another example is:

```
  group1,!group2
```

which, when evaluated as an `AND` expression, will be `TRUE` if the current
user is a member of "group1" **and not** a member of "group2".

### `OR` Expressions

`OR` expressions are simpler, for only _one_ element in the list needs to
evaluate to `TRUE` for the entire expression to be `TRUE`.  These expressions
are usually useful for user and class names.  For example:

```
  user1,user2
```

when evaluated as an `OR` expression, will be TRUE if the current user is
either "user1" **or** "user2".  However, using the `!` negation notation in an
`OR` expression can easily catch the administrator offguard:

```
  user2,!user3
```

As an `OR` expression, this would evaluate to `TRUE` for "user1", for
`user1 != user3` is `TRUE`.  This logic applies to class name `OR` expressions,
too.

There _are_ times when the developer may want to use user or class names
in `AND` expressions, _e.g._:

```
  !user2,!user3
```

would evaluate to `TRUE`, as an `AND` expression, for any user _other than_
"user2" and "user3".

### Expression API

The ProFTPD API functions for performing the evaluation of a given expression
are:

* `pr_class_and_expression()`
* `pr_class_or_expression()`
* `pr_group_and_expression()`
* `pr_group_or_expression()`
* `pr_user_and_expression()`
* `pr_user_or_expression()`

Note that for class expressions to be functional, the tagging of a client
session with a class _must_ be enabled via the use of `<Class>` sections.
The above functions all use the pertinent [session](session.md) members
for comparisons, and those mmembers only became valid for use _after_ the
client has connected and authenticated.

---

[Table of Contents](../toc.md)

---
