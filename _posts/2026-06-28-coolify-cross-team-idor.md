---
layout: post
title: "CVE-2026-57498: the API scoped everything to your team, the UI did not"
summary: "Coolify's API controllers filter every server lookup by team. Seven Livewire components took the same identifiers from the query string and looked them up unscoped."
reading_time: "6 min"
tags: [cve, idor, multi-tenancy, php]
lang: en
ref: coolify-cross-team-idor
cve: CVE-2026-57498
---

**CVE-2026-57498** · [GHSA-725v-f5gh-22q9](https://github.com/coollabsio/coolify/security/advisories/GHSA-725v-f5gh-22q9) ·
Coolify · CWE-639 · Critical, CVSS 9.6 · published 2026-06-28 · fixed in v4.0.0-beta.474

Coolify is a self-hosted deployment platform. Teams own servers, and a team must not be able
to touch another team's infrastructure. That boundary is the product.

The API enforces it consistently:

```php
$server = Server::whereTeamId($teamId)->whereUuid($serverUuid)->first();
```

Every server lookup in the API controllers goes through that scoping. The clone operation in
`ResourceOperations.php` scopes its destination lookup the same way, through a `whereHas` on
the server's team.

The web UI does not.

## The unscoped path

`app/Livewire/Project/Resource/Create.php`:

```php
$destination = StandaloneDocker::whereUuid($destination_uuid)->first();  // no team check
$service_payload = [
    'server_id'      => (int) $server_id,   // straight from the query string
    'destination_id' => $destination->id,
];
$service = Service::create($service_payload);
```

`server_id` and `destination_uuid` arrive as URL query parameters and are used as-is.

What makes this one interesting is that the component *does* validate. `project_uuid` and
`environment_uuid` are both checked against `currentTeam()` a few lines earlier. Somebody
thought about tenancy in this exact function, applied it to two identifiers, and did not
apply it to the other two.

The same shape repeats across seven or more Livewire components (the Docker Compose creator,
the Docker image creator, the private repository flow) and in the `create_standalone_*`
helpers they share.

## Why the UI is the soft side

Two things converge here.

The API is the surface that gets audited. It is documented, it has a schema, people write
clients against it, and its controllers are where a reviewer looks first. The web UI reaches
the same models through a different framework layer, and in a Livewire or Hotwire style
application the component *is* the controller, holding state that arrives from the URL.

And a component that renders a form does not read like an authorisation boundary. It reads
like a view. The identifiers it accepts feel like presentation state, not object references,
right up until one of them ends up in a `create()` call.

So the rule I now apply to any multi-tenant application: find the scoping helper, then list
every reference to the scoped model that does **not** go through it. In Coolify that meant
grepping for `Server::` and `StandaloneDocker::` and reading the ones without a
`whereTeamId` or `whereHas`. The API side was clean. The UI side was not.

## Impact and fix

An authenticated user of any team can deploy resources onto another team's server, and reach
their Docker destinations. On a deployment platform, running a container on infrastructure you
do not own is code execution on that infrastructure, which is what carries this to 9.6 with a
changed scope.

Fixed in v4.0.0-beta.474 by applying the same team scoping the API already used.

## Timeline

- Reported to the maintainers
- Fixed in v4.0.0-beta.474
- Advisory published 2026-06-28, CVE-2026-57498 assigned
