---
layout: post
title: "CVE-2026-41522: a second API that never inherited the first one's authorisation"
summary: "DFIR-IRIS enforced case access carefully in its REST API. An optional GraphQL endpoint exposed the same data with none of those checks, and nobody was using it."
reading_time: "5 min"
tags: [cve, broken-access-control, graphql, python]
lang: en
ref: iris-graphql-authz
cve: CVE-2026-41522
---

**CVE-2026-41522** · [GHSA-3mxh-x92q-9r25](https://github.com/dfir-iris/iris-web/security/advisories/GHSA-3mxh-x92q-9r25) ·
DFIR-IRIS · CWE-285 · High · published 2026-05-28 · fixed in 2.4.28

DFIR-IRIS is an incident response collaboration platform. Its whole model is that analysts see
the cases they are assigned to and not the others, and its REST API enforces that properly:
role checks, case ACLs, the works.

It also shipped an optional GraphQL endpoint at `/graphql`. That endpoint resolves the same
objects through a completely separate resolver layer, and that layer never received the
authorisation model.

Three consequences, all reachable by **any authenticated user** regardless of role or case
assignment:

- `ioc(iocId: …)` resolves an indicator by primary key with no case-access check. Iterate the
  IDs and you read every IOC in the database.
- `case(caseId: …).iocs` returns the indicators attached to any case, without checking access
  to that case.
- The `caseCreate` mutation creates a case without checking the `standard_user` permission.

On an incident response platform, the IOC table is the sensitive object. It is what is being
investigated, on whose infrastructure, and how far the responders have got.

## Two doors, one lock

This is the clearest example in my set of a pattern worth internalising: **when a product
exposes the same data through two APIs, authorisation is almost never shared between them.**

The REST layer accumulated its checks over years, one incident at a time. The GraphQL layer
was added later, wired to the ORM directly, and resolvers that hit the database by primary key
are the default shape in every GraphQL tutorial. Nothing in that layer imports the REST
permission model, so nothing in it enforces it.

The same asymmetry shows up between a web UI and its API, between v1 and v2 of an API, and
between a REST surface and a gRPC or WebSocket one. Whenever you find two entry points onto
one data model, the question is not whether they agree, it is **where** they diverge.

For a black-box audit the tell is cheap to check: request an object through the well-worn API,
get denied, then request the same object through the newer surface and compare.

## The fix, which I did not expect

The maintainers shipped 2.4.28 by **removing** the GraphQL blueprint, its resolvers and its
dependencies entirely. The feature was not in use.

That is the right call and it is worth naming. An unused optional feature is pure attack
surface: it has all the exposure of a supported feature and none of the scrutiny, because
nobody is looking at code nobody calls. Deleting it costs nothing and removes the entire
class of issue permanently.

For anyone who cannot upgrade immediately, the advisory's workaround is to block `/graphql` at
the reverse proxy.

## Impact and fix

Any authenticated user could read every indicator of compromise in the platform, enumerate
indicators per case, and create cases without the required permission.

Fixed in 2.4.28. Affected: 2.4.27 and earlier.

## Timeline

- Reported to the maintainers
- GraphQL endpoint removed in 2.4.28
- Advisory published 2026-05-28, CVE-2026-41522 assigned
