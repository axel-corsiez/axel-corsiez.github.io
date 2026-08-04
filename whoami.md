---
layout: post
title: "whoami"
permalink: /whoami/
lang: en
alt_url: /fr/whoami/
---

Offensive security researcher. I spend most of my time reading other people's code looking for
the places where a security control exists but does not cover everything it should.

That sounds narrow, and it is deliberately narrow. Almost none of the eleven published
advisories credited to me are "the developer forgot about security". They are the opposite: a
guard was written, correctly, and then one code path was left outside it. A sanitiser sixty
lines above the endpoint that does not call it. An authentication decorator applied one line
too high so the framework silently discards it. An allowlist covering the core configuration
writer but not the plugin one. Finding those is a matter of establishing what the majority
spelling is in a codebase, and then reading the minority.

## What I work on

**Vulnerability research on open-source software.** Source-code auditing, variant analysis on
published fixes, and coordinated disclosure. Eleven published advisories to date, spanning
authentication bypass, unauthenticated path traversal, SSRF, cross-tenant authorisation and
command execution. They are listed with their write-ups on [the index]({{ '/' | relative_url }}).

**Tooling.** I built the pipeline that makes the above possible at scale rather than one
repository at a time, and I write about how it works and where it fails.

**Web application security.** Access control, server-side request forgery, injection,
authentication and session handling. This is where most of my findings land.

## How I work

A finding is not a finding until a proof of concept runs. No conditional language, no "an
attacker could potentially". If it does not reproduce on a real deployment with realistic
data, it does not get reported, and if it reproduces but grants nothing of consequence, it
still does not get reported: a technically real bug with no impact wastes a maintainer's
afternoon.

Reports go out with the working proof of concept and a concrete remediation, because the
objective is a patch and not a trophy. Advisories get published when the project is ready,
not when it would be most interesting to post.

## Disclosure

Everything published here concerns open-source code analysed locally, or systems I am
authorised to test. Findings go to maintainers first and are written about only once fixed.
Work that is still in review appears on this site as a count, with no project named.
