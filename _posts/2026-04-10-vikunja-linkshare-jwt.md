---
layout: post
title: "CVE-2026-35594: revoking a share that cannot be revoked"
summary: "Vikunja built link-share authorisation entirely from JWT claims, with zero database lookups. Deleting the share removed the row and changed nothing for anyone already holding a token."
reading_time: "6 min"
tags: [cve, authorization, jwt, go]
lang: en
ref: vikunja-linkshare-jwt
cve: CVE-2026-35594
---

**CVE-2026-35594** · [GHSA-96q5-xm3p-7m84](https://github.com/advisories/GHSA-96q5-xm3p-7m84) ·
Vikunja · CWE-613 · Medium, CVSS 6.5 · published 2026-04-10 · fixed in 2.3.0

Vikunja lets you share a project through a link. You pick a permission level, you get a URL,
you send it to someone. Later you delete the share, or you downgrade it from admin to
read-only.

That second action did not do what the interface implied.

## Authorisation with no database

Link-share requests authenticate with a JWT. The function that turns those claims back into a
share object is `GetLinkShareFromClaims`, and it performs **zero queries**. It reads `id`,
`hash`, `project_id`, `permission` and `sharedByID` straight out of the token and builds the
struct from them.

That struct then goes directly into the permission checks:

| check | database queries |
|---|---|
| `GetLinkShareFromClaims` | 0 |
| `Project.CanRead` (link share) | 0 |
| `Project.CanWrite` (link share) | 0 |
| `Project.IsAdmin` (link share) | 0 |

Nothing in that path ever asks whether the share still exists, or what its permission is
*now*. The token is not a pointer to a record. The token **is** the record.

So deleting a link share removes a row that authorisation never consults, and downgrading a
share rewrites a column that authorisation never reads. Anyone already holding a token keeps
the original permission level until the token expires on its own.

## The window

That expiry is the second half of the problem. Vikunja uses two different TTLs:

- user tokens: **10 minutes**, with a `sid` claim and a server-side session, so they can be revoked
- link-share tokens: **72 hours**, no `sid`, no server-side session, no refresh mechanism

The short-lived, revocable token got the careful treatment. The long-lived, non-revocable one
got three days.

Three days is a long time to keep admin on a project you believed you had un-shared. The
realistic scenario is not an attacker, it is an ordinary user: a contractor whose access was
ended, a link posted somewhere it should not have been and then deleted, a permission level
lowered after someone did something they should not have. In each case the operator performs
the corrective action, sees it succeed, and remains exposed.

## Why it is worth reporting a medium

It scores 6.5. It needs an already-issued token, and it expires by itself. I reported it
anyway, because the severity number describes the exploit and not the failure.

The failure is that a security control in the product does not do the thing its name says.
"Delete share" that does not end access is worse than no delete button, because it converts
an unresolved risk into one the operator believes they have handled. That gap between the
mental model and the behaviour is the actual bug, and no CVSS vector has a field for it.

The structural lesson is easy to reuse: when a system authenticates from claims, ask what
happens on **revocation**, not on forgery. Signature verification is usually solid. Revocation
is where stateless designs quietly decide that a permission decision made in the past applies
to the present.

## Impact and fix

Up to 72 hours of continued access at the original permission level after a share is deleted
or downgraded.

The fix, shipped in 2.3.0, is to look the share up server-side and validate the current
permission on every request, instead of trusting the level baked into the token.

## Timeline

- Reported to the maintainers
- Fixed in 2.3.0
- Advisory [GHSA-96q5-xm3p-7m84](https://github.com/advisories/GHSA-96q5-xm3p-7m84) published on 2026-04-10
- CVE-2026-35594 assigned
