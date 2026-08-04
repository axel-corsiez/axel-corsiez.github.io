---
layout: post
title: "CVE-2026-35490: authentication that was never in the call chain"
summary: "Thirteen Flask routes had the auth decorator written above @route instead of below it. The code looked protected, read as protected in review, and was not protected at all."
reading_time: "6 min"
tags: [cve, authentication-bypass, flask, python]
lang: en
ref: changedetection-decorator-order
cve: CVE-2026-35490
---

**CVE-2026-35490** · [GHSA-jmrh-xmgh-x9j4](https://github.com/advisories/GHSA-jmrh-xmgh-x9j4) ·
changedetection.io · CWE-863 · Critical · published 2026-04-06 · fixed in 0.54.8

This one is my favourite of the batch, because nothing is missing. The authentication
decorator is there. It is spelled correctly. It is the right decorator. It is simply written
one line too high.

## Two lines, opposite outcomes

The correct order, used on more than thirty routes in the project:

```python
@blueprint.route('/settings', methods=['GET'])
@login_optionally_required
def settings():
    ...
```

The order used on thirteen routes across five blueprint files:

```python
@login_optionally_required
@blueprint.route('/backups/download/<filename>')
def download_backup(filename):
    ...
```

Decorators in Python apply bottom-up. `@route()` does not wrap a function, it **registers**
the function it receives with the URL map and hands it back. So it has to be outermost, which
means topmost.

Reverse the two and `@route()` receives the raw view function and registers *that* with the
URL map. The auth wrapper is then applied to whatever `@route()` returned, and the result is
bound to a module-level name nobody ever calls. Flask happily serves the unprotected original.

No exception, no warning, no log line. The wrapper exists and is simply never in the call
chain.

## Reading it from the outside

The behaviour is unambiguous from a black-box position, which is what makes it easy to prove.
With a password configured:

```
GET /            -> 302 /login?next=/
GET /settings    -> 302 /login
```

Authentication is on. Then, with no session cookie at all:

```
GET /backups/request-backup  -> 302 /backups/     (not /login: the backup was created)
GET /backups/                -> 200               (listing)
```

A redirect to the resource instead of to the login page is the whole tell. The affected routes
covered backup creation, listing and download, which on this application means the watch
configuration and its history.

## What made it findable

The project was doing the right thing everywhere else. Thirty-odd routes with the correct
order, thirteen with it reversed, all in the same codebase, all written by people who clearly
understood the decorator.

That ratio is the signal. A codebase that never protects anything is a design problem; a
codebase that protects thirty-seven things out of fifty has a *consistency* problem, and
consistency problems are mechanically detectable. You do not need to understand the
application. You need to establish which spelling is the majority and then look at the
minority.

This is the same detector that surfaced the pyLoad issue in the same week, pointed at a
different framework: find the guard, enumerate every place it should appear, diff.

Note also the failure mode. This is not "the developer forgot authentication". It is
"the developer wrote authentication and the language quietly discarded it". Code review does
not catch it, because a reviewer scanning for the presence of `@login_optionally_required`
finds it present. Only order matters, and order is exactly what the eye skips.

## Impact and fix

Unauthenticated access to thirteen routes on an application whose entire purpose is to store
what you are monitoring and what changed. The fix is a one-line move on each route, and the
durable version is a test that asserts every registered endpoint resolves to a wrapped view.

## Timeline

- Reported to the maintainer
- Fixed in 0.54.8
- Advisory [GHSA-jmrh-xmgh-x9j4](https://github.com/advisories/GHSA-jmrh-xmgh-x9j4) published on 2026-04-06
- CVE-2026-35490 assigned
