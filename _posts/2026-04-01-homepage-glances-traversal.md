---
layout: post
title: "GHSA-rg3r-jprv-xq38: path traversal that takes your credentials with it"
summary: "Homepage interpolated a version number from the query string straight into an outbound URL, and forwarded the stored Basic Auth header to wherever that URL pointed."
reading_time: "5 min"
tags: [advisory, path-traversal, ssrf, javascript]
lang: en
ref: homepage-glances-traversal
---

[GHSA-rg3r-jprv-xq38](https://github.com/advisories/GHSA-rg3r-jprv-xq38) · Homepage
(`gethomepage/homepage`) · CWE-22 · Medium, CVSS 4.3 · published 2026-04-01 · fixed in 1.12.3

Homepage is the dashboard a lot of people put in front of their self-hosted stack. Its widgets
proxy requests to backend services, which means it stores credentials for those services and
attaches them to outbound calls.

The Glances widget handler builds its URL like this:

```js
`${url}/api/${privateWidgetOptions.version}/${endpoint}`
```

`version` comes from `req.query.version`, with a nullish fallback to `3` and no other
handling. Supply `version=3/../../anything` and the outbound request walks out of the API
namespace to a path of your choosing on the Glances backend.

The part that raises this above a curiosity: the handler attaches the widget's stored Basic
Auth credentials to that request. So the traversal does not just reach an unintended path, it
reaches it **authenticated**.

## The secondary finding

The widget has an endpoint allowlist:

```js
allowedEndpoints: /\d\/quicklook|diskio|cpu|fs|gpu|system|mem|network|processlist|sensors|containers/
```

Alternation in a regular expression has the lowest precedence, so this does not read as
"a digit, a slash, then one of these endpoints". It reads as `\d\/quicklook` **or** `diskio`
**or** `cpu`, and so on, each as an independent unanchored alternative. Without anchors, any
string containing `cpu` anywhere matches.

It is broken, but it is also irrelevant here, because the custom Glances handler never
consults it. I reported both anyway: a filter that does not do what it looks like it does is
worth fixing before someone starts relying on it.

## Why I like this one

It is a small finding, 4.3, no CVE assigned. I include it because it shows the two habits that
find most of this class.

The first is treating **every interpolated value in an outbound URL** as attacker input,
including the ones that look structural. Nobody thinks of an API version as user data. It sits
in the path where a constant should be, and that is precisely why it gets no validation.

The second is asking what travels **with** the request. A proxy handler that forwards
credentials turns a mild traversal into a credentialed one, and the severity of the
traversal is decided by what the header carries, not by the path itself.

## Impact and fix

Path traversal on the backend Glances instance from the dashboard, with the dashboard's stored
credentials attached, plus a non-functional endpoint allowlist.

Fixed in 1.12.3.

## Timeline

- Reported to the maintainers
- Fixed in 1.12.3
- Advisory published 2026-04-01
