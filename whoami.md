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

**Bug bounty on live targets.** Same reflexes, different visibility: no source, only what the
target exposes. Recon and attack-surface mapping, JavaScript bundle mining for endpoints the
interface never calls, and access-control testing driven with two controlled identities so an
authorisation gap is demonstrated rather than asserted. In scope, on programs that authorise it,
never with another user's data.

**Tooling.** I built the pipeline that makes the above possible at scale rather than one
repository at a time, and I write about how it works and where it fails.

**Web application security.** Access control, server-side request forgery, injection,
authentication and session handling. This is where most of my findings land.

## Certifications

<ul class="certs">
  <li><b>OSCP</b>OffSec Certified Professional <span class="dim">(PEN-200)</span></li>
  <li><b>OSWA</b>OffSec Web Assessor <span class="dim">(WEB-200)</span></li>
</ul>

## Why open source

The software people self-host is maintained by teams of two or three. They are good engineers
and they are not security engineers, they have no budget for an audit, and the projects are big
enough that nobody holds the whole thing in their head. Meanwhile the code runs the internal
tooling of companies that would never ship it themselves.

That gap is where I work. Eleven advisories, ten fixed releases shipped by their maintainers,
all of it free for the project and permanent for every deployment that updates. It is the
highest-leverage security work I know of, and the reason none of it is behind a paywall here.

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

## Contact

<ul class="contact">
  <li><b>github</b><a href="https://github.com/axel-corsiez" rel="me noopener">axel-corsiez</a></li>
  <li><b>linkedin</b><a href="https://www.linkedin.com/in/axel-corsiez/" rel="me noopener">axel-corsiez</a></li>
  <li><b>mail</b><a href="{{ site.email_href }}">{{ site.email_text }}</a></li>
</ul>

<p class="avail"><b>Open to opportunities.</b> Vulnerability research, application security, offensive tooling.</p>
