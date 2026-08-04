---
layout: post
title: "Hunting CVEs at scale: notes from auditing 83 open-source projects"
summary: "Why I stopped reading code looking for bugs, and started looking for bug-shaped code instead. The design of a research pipeline, and what it actually returns."
reading_time: "8 min"
tags: [vulnerability-research, tooling, static-analysis, methodology]
---

Reading one codebase properly takes days. Reading eighty takes a year you do not have.

That constraint is the whole story behind the pipeline I have been building for the past
months. It is not an attempt to replace human judgment. It is an attempt to spend that
judgment on the twenty candidates that deserve it, instead of on the twenty thousand lines
that do not.

This article describes the shape of the system and what it returns in practice. It
deliberately stays away from specifics: several findings are still moving through
coordinated disclosure, and nothing unpatched is described here.

## The premise: a fix is a confession

The most useful realisation was that maintainers tell you where the weak code is, after
the fact, in their commit history.

When a project patches a security issue, the commit does two things. It closes one hole,
and it publishes the *shape* of that hole: which helper was missing, which value was
trusted, which normalisation step was skipped. That shape is almost never unique to the
file it was found in. Large codebases repeat themselves. The same developer, the same
idiom, the same wrong assumption, three directories away.

So the first engine does not look for vulnerabilities at all. It reads the last N days of
security-relevant commits, extracts the pattern of what was fixed, and then goes hunting
for **siblings that were not patched**. Variant analysis, automated, aimed at the one
place where the odds are genuinely in your favour.

The second thing that history gives you is new attack surface. Code that landed last month
has had a month of review. Code that landed last year has had a year. Recent features are
systematically softer, and the diff tells you exactly which ones they are.

## Not every target deserves the same pipeline

An early mistake was treating every repository identically. A reverse proxy and a
project-management app fail in completely different ways, and running the same checks
against both wastes time on one and misses the other.

So targets get classified first, roughly into application-layer, infrastructure-layer, and
the hybrids that most large platforms actually are. That label routes which engines run.
Data-flow analysis is worth its cost on an application handling user objects and
permissions. On a proxy, the interesting failures live in header canonicalisation, parser
disagreements and trust boundaries, and the same analysis mostly returns noise.

## Three engines, one shortlist

Beyond the diff engine, two others run and are merged:

**A pattern knowledge base.** Seventeen vulnerability classes are modelled explicitly:
header canonicalisation, request smuggling, path normalisation, trusted-IP handling,
certificate verification, token algorithm confusion, deserialisation, archive extraction,
upload handling, object-level authorisation, injection, and more. Each is a description of
what the mistake *looks like* in real code across the frameworks that matter, not a regex
hoping for a lucky hit.

**A taint engine.** Classic source-to-sink reachability: can attacker-controlled input
arrive somewhere dangerous. The part that took the longest to get right was not finding
flows. It was *grading* them.

## The thing that actually matters: scoring, not booleans

A scanner that flags everything is worse than no scanner. You stop reading its output
within a week, and then it is just a process burning CPU.

So nothing in the pipeline emits a binary verdict. Two independent gradings decide whether
a candidate is worth a human's time:

- **How well is the flow proven?** Eight levels, from a direct source-to-sink call in one
  function, down through helpers and object propagation, to "the source and the sink are
  in the same file and nothing connects them." Only the top of that scale earns attention.
- **How real is the validation in the way?** Genuine validation, such as parsing to an
  integer or reducing to a basename, collapses a candidate's score. Cosmetic validation
  that only looks protective, like trimming whitespace, barely moves it. This distinction
  alone killed most of the false positives.

Then the survivors are checked against what the world already knows, meaning public
advisory databases and bounty platforms, and split into *novel* and *known-class*.
Rediscovering a documented issue is not research, and it is not worth reading.

Only what comes out of that funnel gets human and model triage. The pipeline has its own
regression tests, covering direct flows, cross-function propagation, the validation
grading and, importantly, false-positive rejection. Every change has to keep proving it
still refuses to cry wolf.

<div class="note">
  <span class="note-label">On the LLM question</span>
  <p>
    A language model is used, but at the end and on a shortlist, as a reviewer of ranked
    candidates, never as the thing that scans the repository. Pointed at a whole codebase
    it hallucinates plausible vulnerabilities with total confidence. Pointed at twenty
    pre-scored candidates with the data-flow already established, it is genuinely useful.
    The engineering is in what you refuse to hand it.
  </p>
</div>

## What it actually returns

Across the corpus so far: **83 open-source projects processed, 35 taken far enough to
produce a written report.** Self-hosted platforms, LLM tooling, ITSM and asset management,
CI/CD, developer infrastructure. Mostly projects in the range where the code is big enough
to hide things but the security team is small or nonexistent.

A small number of advisories have gone out through coordinated disclosure. None are
published yet, which is exactly why none are described here. The projects get their window
first, and the write-ups follow the fix, not the finding.

The ratio is worth stating plainly, because tooling articles usually hide it: most targets
return nothing worth reporting. That is the correct outcome. A pipeline that produces a
finding on every project is a pipeline producing fiction.

## What it does not do

Being honest about the limits is what makes the rest credible:

- It finds bug **classes**. Logic flaws that need real semantic understanding of what the
  application is *for* still walk straight past it.
- Memory-corruption and parser-crash bugs are out of scope entirely. That is fuzzing work,
  a different discipline with different tools.
- On very mature, heavily audited targets, nearly everything it surfaces is already known
  or too weak to exploit. Expected, and the dedup stage is there precisely to say so
  quickly instead of wasting a day.

## Disclosure

Everything here is done against open-source code, on infrastructure I control, and
reported to maintainers before it is written about. Reports go out with a working proof of
concept and a concrete remediation, and the advisory gets published when the project is
ready, not when it would be most interesting to post.

Follow-up articles will go into individual cases, starting with the ones already through
that process.
