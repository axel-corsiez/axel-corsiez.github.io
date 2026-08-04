---
layout: post
title: "CVE-2026-34586: the share expired everywhere except where the file is served"
summary: "PdfDing checked expiry, view limits and deletion on the viewer page. The two endpoints that actually hand over the PDF checked only that a session existed."
reading_time: "5 min"
tags: [cve, broken-access-control, django, python]
lang: en
ref: pdfding-share-revocation
cve: CVE-2026-34586
---

**CVE-2026-34586** · [GHSA-vfqx-2464-38wf](https://github.com/mrmn2/PdfDing/security/advisories/GHSA-vfqx-2464-38wf) ·
PdfDing · CWE-863 · Medium, CVSS 6.5 · published 2026-03-30 · fixed in v1.7.1

PdfDing is a self-hosted PDF manager. You can share a document with an expiry date, a maximum
number of views, and you can delete the share afterwards. Three separate ways to end access.

The viewer page honours all three. `ViewShared.get()` checks `inactive` (which covers expiry
and the view cap) and `deleted` before rendering anything.

Two other endpoints do not:

- `/pdf/shared/get/<id>/<revision>` serves the file
- `/pdf/shared/download/<id>` downloads the file

Both route through `check_shared_access_allowed()`, and that function validates one thing:
that a session exists. Not whether the share expired. Not whether the view limit was reached.
Not whether it was deleted.

So the page that shows you the document respects the rules, and the endpoints that hand you
the bytes do not.

## The gap between the wrapper and the resource

This is the same failure as the Vikunja link-share issue I wrote up
[here]({{ '/2026/04/vikunja-linkshare-jwt/' | relative_url }}), arriving from a different
direction. There, revocation was ignored because authorisation was rebuilt from a token. Here,
revocation is checked correctly, but only on the HTML page, while the file itself is served by
a different handler with a weaker check.

The user-visible behaviour is what makes it worth reporting. The owner sets an expiry, or hits
delete, and the interface confirms it. Anyone who opened the share before that keeps a working
session, and a working session is all the serve endpoint asks for. They can keep pulling the
document indefinitely.

There is a smaller sibling in the same advisory: `ViewShared.post()` checks `inactive` but not
`deleted`, so a **new** session can still be created for a soft-deleted share.

## What to check for this class

When an application has a resource behind a viewer, ask which handler actually produces the
bytes. It is almost never the one you are looking at. Thumbnails, downloads, direct-serve
routes, range requests and export endpoints tend to be written later, by someone optimising
delivery, and they frequently reimplement a thinner version of the access check.

The question to ask of any access-control helper is not "is it called" but "is it called by
**every** path that reaches the resource, and does it check every condition the product
promises". Here it was called everywhere. It just did not check enough.

## Impact and fix

Continued access to shared documents after expiry, after the view limit, and after deletion,
for anyone who had opened the share once.

Fixed in v1.7.1 by validating `inactive` and `deleted` inside the shared access check itself,
so every consumer inherits it.

## Timeline

- Reported to the maintainer
- Fixed in v1.7.1
- Advisory published 2026-03-30, CVE-2026-34586 assigned
