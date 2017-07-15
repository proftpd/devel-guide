# ProFTPD Developer Guide

---

[Table of Contents](toc.md)

---

## Purpose

For any given program or application, almost every user or developer poses
questions of "How do I get _X_ program to do _Y_ nifty thing?" The ProFTPD
server is structured similar to the Apache HTTP server, and can be _extended_
through modules, much like Apache.  Consequently, this extending or hacking-in
of new functionality and capabilities is most easily done by developing a
module, rather than changing the source code itself.

ProFTPD modules are designed to handle and respond to configuration
_directives_ at startup, to FTP commands sent over the control connection by
FTP clients, and to authentication requests.  Here is a more in-depth
[introduction](introduction.md) to ProFTPD modules, and how they tick.

Unless otherwise stated, this information is relevant for the current version
of ProFTPD.  Any opinions, ideas, and philosophies contained herein are my
own, or those of the contributors, and do not reflect the ideas or
philosophies of my employer.

## Audience

This manual is intended for system administrators and programmers who wish to
develop modules and patches for the ProFTPD server.  Readers should be
comfortable programming in a Unix environment, be familiar with C, and be aware
of how the FTP protocol works.  These documents aim to provide an explanation
of the inner workings of ProFTPD and its modules, and how to develop them.
Readers should therefore have in interest in these topics, else they may be
bored.

## Additions and Contributions

The current manual is not yet complete. I will continue to work on it as I find
time.  Feel free to send in material for the "TODO" sections, and for the small
notes I have left around. Also, compatibility is an issue I struggle with
sometimes. The best I can do for some Unix flavors is read man pages.
Corrections, addendums, and notes for compatibility are greatly appreciated.
All contributions, comments, and criticisms are welcome, and can be sent to
_tj@castaglia.org_.

## Copyright

I, TJ Saunders, reserve a collective copyright on this manual.  Individual
contributions made to this manual are the intellectual property of the
contributor.

This manual may contain errors, or inaccurate material. Use it at your own risk.
Although an effort is made to keep all the material presented here accurate,
the contributors and maintainer of these documents will not be held
responsible for any damage, _direct or indirect_, which may result from
inaccuracies.

This document may be reproduced in whole or in part, without fee, subject to
the following restrictions:

* The copyright notice above and this permission notice must be preserved
  complete on all complete or partial copies.
* Any translation or derived work must be approved by the author in writing
  before distribution.
* If you distribute this work in part, instructions for obtaining the complete
  version of this manual must be included, and a means for obtaining a complete
  version provided.
* Small portions may be reproduced as illustrations for reviews or quotes in
   other works without this permission notice if proper citation is given.

Exceptions to these rules may be granted for academic purposes: write to the
author and ask. These restrictions are here to protect us as authors, not to
restrict you as learners and educators.

## Comments

One final important thing to note: this documentation is an attempt to explain
what the actual code is doing; the only true canonical source for answers to
any questions that might arise is the source code itself.  This documentation
may, and most likely will, fall out of date, or not properly reflect what the
code is doing.  Therefore, the author of these documents should not be held
responsible if ProFTPD is not behaving exactly as described here.  Instead,
s/he should be held responsible for not describing things exactly as ProFTPD
is behaving.  As always, _caveat emptor_.

---

[Table of Contents](toc.md)

---
