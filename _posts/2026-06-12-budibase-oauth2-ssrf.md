---
layout: post
title: "CVE-2026-48146: one outbound call out of a dozen that skipped the SSRF wrapper"
summary: "Budibase routes every outbound request through a blacklist-checking fetch. Every one except the OAuth2 token endpoint, which called the raw fetch directly."
reading_time: "6 min"
tags: [cve, ssrf, typescript, variant-analysis]
lang: en
ref: budibase-oauth2-ssrf
cve: CVE-2026-48146
---

**CVE-2026-48146** · [GHSA-g6qx-g4pr-92v7](https://github.com/advisories/GHSA-g6qx-g4pr-92v7) ·
Budibase (`@budibase/server`) · CWE-918 · High, CVSS 7.7 · published 2026-06-12 · fixed in
3.39.0

Budibase is a low-code platform. Users build applications, and those applications talk to
things: webhooks, APIs, object storage, OAuth2 providers. Outbound requests are the product,
which means server-side request forgery is a threat the project has clearly thought about.

It has a wrapper for exactly that:

```typescript
export async function fetchWithBlacklist(url: string, opts?: RequestInit) {
  await blacklist.isBlacklisted(url)          // rejects internal ranges
  const response = await fetch(url, { ...opts, redirect: "manual" })
  // re-checks every redirect target
}
```

Two details worth noting, because they show the author understood the problem properly.
`redirect: "manual"` disables automatic following, and each redirect target is re-checked.
Validating only the first URL and letting the HTTP client chase a 302 into the internal
network is the classic way an SSRF filter gets bypassed, and this wrapper does not make that
mistake.

## The one that got away

`fetchWithBlacklist` is used for the Discord webhook step, the Slack step, the Make.com
integration, plugin downloads, object store access. Consistently, across the codebase.

Then, in `packages/server/src/sdk/workspace/oauth2/utils.ts`:

```typescript
async function fetchToken(config: OAuth2Config): Promise<TokenResponse> {
  // ...
  const response = await fetch(config.url, fetchConfig)   // raw fetch
  // ...
}
```

`config.url` is the OAuth2 token endpoint, and it is configuration a user with the BUILDER
role supplies. Point it at an internal address, save the configuration, trigger a validation,
and the server makes the request on your behalf.

## Why the "validate the config" button is a good target

There is a reason this specific call was the one that slipped, and it generalises.

Most outbound calls in an application are obviously outbound: this step sends a message to
Slack, that step downloads a plugin. They read as network operations, so whoever wrote them
reached for the network helper.

`fetchToken` does not read as a network operation. It reads as **validation**. Its job is to
confirm the credentials work, and the request is incidental to that. That framing is enough
for the safe wrapper to not come to mind.

So when I audit an application that has an outbound-fetch allowlist, I stop grepping for the
places that clearly make requests, and start looking for the ones where a request is a
side effect of something else: connection tests, "verify configuration", health checks, preview
and thumbnail generation, metadata fetches, SSO discovery documents. That is where the wrapper
goes missing.

## Impact and fix

A BUILDER-role user reaches internal services the deployment does not expose: in Budibase's own
architecture that includes CouchDB, and in a cloud deployment it includes the instance metadata
endpoint. Scope changes because the request crosses a trust boundary, which is what carries the
CVSS to 7.7 despite requiring a privileged role.

The fix in 3.39.0 is one line in spirit: route the OAuth2 token fetch through
`fetchWithBlacklist` like every other outbound call.

## Timeline

- Reported to the maintainers
- Fixed in 3.39.0
- Advisory [GHSA-g6qx-g4pr-92v7](https://github.com/advisories/GHSA-g6qx-g4pr-92v7) published on 2026-06-12
- CVE-2026-48146 assigned
