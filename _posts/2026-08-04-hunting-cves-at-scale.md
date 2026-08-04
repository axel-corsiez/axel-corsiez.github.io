---
layout: post
title: "Hunting CVEs at scale: notes from auditing 82 open-source projects"
summary: "Why I stopped reading code looking for bugs, and started looking for bug-shaped code instead. The architecture of a research pipeline, and what it actually returns."
reading_time: "16 min"
tags: [vulnerability-research, tooling, static-analysis, methodology]
lang: en
ref: cve-pipeline
---

Reading one codebase properly takes days. Reading eighty takes a year you do not have.

That constraint is the whole story behind the pipeline I have been building for the past
months. It is not an attempt to replace human judgment. It is an attempt to spend that
judgment on the twenty candidates that deserve it, instead of on the twenty thousand lines
that do not.

This article describes the architecture and what it returns in practice. It deliberately
stays away from specifics: several findings are still moving through coordinated
disclosure, and nothing unpatched is described here.

<div class="toc-wrap" markdown="1">
<span class="toc-label">contents</span>

* TOC
{:toc}

</div>

## The shape of the problem

Manual auditing does not fail because it is inaccurate. It fails because it is linear. You
read a router file, then a controller, then a helper, and four hours later you have covered
maybe two percent of an application and you are the least sharp you will be all day.

Meanwhile the interesting property of the ecosystem is that the same mistakes recur across
thousands of projects. The industry keeps rebuilding the same upload handler, the same
proxy fetch, the same permission check, and keeps getting them wrong in the same handful
of ways.

That asymmetry is exploitable. If mistakes repeat, they can be described. If they can be
described, they can be searched for. And if you can search for them across a corpus rather
than a file, the economics invert: instead of one deep pass over one project, you get a
shallow pass over eighty and a deep pass over the twenty places that lit up.

## The pipeline, end to end

<figure class="fig">
<div class="fig-frame">
<svg viewBox="0 0 660 830" role="img" aria-label="Diagram of the research pipeline from target repository through classification, three analysis engines, merge and dedup, ranked shortlist, model triage, human verification and coordinated disclosure.">
  <defs>
    <marker id="ah" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 z" class="d-fill-l"/>
    </marker>
    <marker id="aha" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 z" class="d-fill-a"/>
    </marker>
  </defs>

  <rect class="d-box" x="220" y="16" width="220" height="42" rx="2"/>
  <text class="d-t" x="330" y="42" text-anchor="middle">target repository</text>
  <line class="d-line" x1="330" y1="58" x2="330" y2="80" marker-end="url(#ah)"/>

  <rect class="d-box" x="200" y="86" width="260" height="54" rx="2"/>
  <text class="d-t" x="330" y="108" text-anchor="middle">target classifier</text>
  <text class="d-t-s" x="330" y="126" text-anchor="middle">app_layer / infra_layer / hybrid</text>

  <line class="d-line" x1="330" y1="140" x2="330" y2="160"/>
  <line class="d-line" x1="114" y1="160" x2="546" y2="160"/>
  <line class="d-line" x1="114" y1="160" x2="114" y2="180" marker-end="url(#ah)"/>
  <line class="d-line" x1="330" y1="160" x2="330" y2="180" marker-end="url(#ah)"/>
  <line class="d-line" x1="546" y1="160" x2="546" y2="180" marker-end="url(#ah)"/>

  <rect class="d-box" x="16" y="186" width="196" height="84" rx="2"/>
  <text class="d-t" x="114" y="210" text-anchor="middle">diff-first scan</text>
  <text class="d-t-s" x="114" y="230" text-anchor="middle">recent security commits</text>
  <text class="d-t-s" x="114" y="246" text-anchor="middle">unpatched siblings</text>
  <text class="d-t-s" x="114" y="262" text-anchor="middle">new attack surface</text>

  <rect class="d-box" x="232" y="186" width="196" height="84" rx="2"/>
  <text class="d-t" x="330" y="210" text-anchor="middle">pattern knowledge base</text>
  <text class="d-t-s" x="330" y="230" text-anchor="middle">22 modelled classes</text>
  <text class="d-t-s" x="330" y="246" text-anchor="middle">routed by target label</text>
  <text class="d-t-s" x="330" y="262" text-anchor="middle">framework aware</text>

  <rect class="d-box" x="448" y="186" width="196" height="84" rx="2"/>
  <text class="d-t" x="546" y="210" text-anchor="middle">taint engine</text>
  <text class="d-t-s" x="546" y="230" text-anchor="middle">source to sink</text>
  <text class="d-t-s" x="546" y="246" text-anchor="middle">cross-function</text>
  <text class="d-t-s" x="546" y="262" text-anchor="middle">app / hybrid only</text>

  <line class="d-line" x1="114" y1="270" x2="114" y2="292"/>
  <line class="d-line" x1="330" y1="270" x2="330" y2="292"/>
  <line class="d-line" x1="546" y1="270" x2="546" y2="292"/>
  <line class="d-line" x1="114" y1="292" x2="546" y2="292"/>
  <line class="d-line" x1="330" y1="292" x2="330" y2="316" marker-end="url(#ah)"/>

  <rect class="d-box" x="180" y="322" width="300" height="60" rx="2"/>
  <text class="d-t" x="330" y="346" text-anchor="middle">merge + deduplicate</text>
  <text class="d-t-s" x="330" y="366" text-anchor="middle">against OSV / GHSA / huntr</text>

  <line class="d-line" x1="330" y1="382" x2="330" y2="402"/>
  <line class="d-line" x1="220" y1="402" x2="440" y2="402"/>
  <line class="d-line-a" x1="220" y1="402" x2="220" y2="424" marker-end="url(#aha)"/>
  <line class="d-line" x1="440" y1="402" x2="440" y2="424" marker-end="url(#ah)"/>

  <rect class="d-box-a" x="130" y="430" width="180" height="44" rx="2"/>
  <text class="d-t-a" x="220" y="457" text-anchor="middle">novel</text>

  <rect class="d-box-d" x="350" y="430" width="180" height="44" rx="2"/>
  <text class="d-t-s" x="440" y="457" text-anchor="middle">known-class (dropped)</text>

  <path class="d-line-a" d="M220,474 L220,496 L330,496 L330,518" marker-end="url(#aha)"/>

  <rect class="d-box" x="180" y="524" width="300" height="44" rx="2"/>
  <text class="d-t" x="330" y="551" text-anchor="middle">ranked shortlist</text>
  <line class="d-line" x1="330" y1="568" x2="330" y2="592" marker-end="url(#ah)"/>

  <rect class="d-box" x="180" y="598" width="300" height="44" rx="2"/>
  <text class="d-t" x="330" y="625" text-anchor="middle">model triage</text>
  <line class="d-line" x1="330" y1="642" x2="330" y2="666" marker-end="url(#ah)"/>

  <rect class="d-box" x="180" y="672" width="300" height="44" rx="2"/>
  <text class="d-t" x="330" y="699" text-anchor="middle">human review + working PoC</text>
  <line class="d-line-a" x1="330" y1="716" x2="330" y2="740" marker-end="url(#aha)"/>

  <rect class="d-box-a" x="180" y="746" width="300" height="44" rx="2"/>
  <text class="d-t-a" x="330" y="773" text-anchor="middle">coordinated disclosure</text>
</svg>
</div>
<figcaption><b>fig. 1</b> The full path from a cloned repository to a report. Everything above the shortlist is automated; everything below it is not.</figcaption>
</figure>

Three properties of this design matter more than any individual component.

**Nothing is trusted enough to be reported automatically.** The pipeline produces
candidates, never findings. The transition from one to the other happens when a human
writes a proof of concept that actually runs.

**Every stage is subtractive.** Each step exists to throw work away, not to add coverage.
That sounds backwards for a research tool, and it is the single most important design
decision in it.

**The expensive stages run last.** Model triage and human review are the scarce resources,
so everything before them is arranged to protect them.

## Choosing what to audit

The corpus is not random. Target selection is its own filter, and getting it wrong wastes
weeks before a single line is scanned.

The sweet spot is projects large enough that nobody can hold the whole codebase in their
head, but not so large that they have a dedicated security team and a decade of prior
audits. In practice that means popular self-hosted software: platforms with real
permission models, file handling, outbound requests and multi-tenancy, maintained by a
handful of people who are excellent engineers and not full-time security engineers.

Signals I weigh:

- **Attack surface density.** Does it upload files, fetch URLs, render templates, handle
  multiple tenants, expose an API? Boring CRUD with no outbound requests is boring to audit.
- **Prior advisory history.** Zero published advisories on a large project usually means
  nobody has looked, not that there is nothing there.
- **Commit velocity in security-adjacent areas.** An active project is generating new
  surface continuously, which is exactly what the diff engine feeds on.
- **Deployment reality.** Something people actually expose to the internet.

## A fix is a confession

The most useful realisation was that maintainers tell you where the weak code is, after
the fact, in their commit history.

When a project patches a security issue, the commit does two things. It closes one hole,
and it publishes the *shape* of that hole: which helper was missing, which value was
trusted, which normalisation step was skipped. That shape is almost never unique to the
file it was found in. Large codebases repeat themselves. The same developer, the same
idiom, the same wrong assumption, three directories away.

<figure class="fig">
<div class="fig-frame">
<svg viewBox="0 0 680 250" role="img" aria-label="Diagram of variant analysis: a security commit is turned into a pattern, the pattern is scanned across the repository, and unpatched siblings come out.">
  <defs>
    <marker id="ah2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 z" class="d-fill-l"/>
    </marker>
    <marker id="aha2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6 z" class="d-fill-a"/>
    </marker>
  </defs>

  <text class="d-t-s" x="6" y="24">the maintainer's own history is the input</text>

  <rect class="d-box" x="6" y="70" width="146" height="66" rx="2"/>
  <text class="d-t" x="79" y="98" text-anchor="middle">security fix</text>
  <text class="d-t-s" x="79" y="118" text-anchor="middle">last N days</text>
  <text class="d-t-s" x="79" y="162" text-anchor="middle">one hole closed</text>

  <line class="d-line" x1="152" y1="103" x2="176" y2="103" marker-end="url(#ah2)"/>

  <rect class="d-box" x="180" y="70" width="146" height="66" rx="2"/>
  <text class="d-t" x="253" y="98" text-anchor="middle">extract shape</text>
  <text class="d-t-s" x="253" y="118" text-anchor="middle">what was wrong</text>
  <text class="d-t-s" x="253" y="162" text-anchor="middle">missing guard,</text>
  <text class="d-t-s" x="253" y="178" text-anchor="middle">trusted value</text>

  <line class="d-line" x1="326" y1="103" x2="350" y2="103" marker-end="url(#ah2)"/>

  <rect class="d-box" x="354" y="70" width="146" height="66" rx="2"/>
  <text class="d-t" x="427" y="98" text-anchor="middle">sweep repo</text>
  <text class="d-t-s" x="427" y="118" text-anchor="middle">same idiom?</text>
  <text class="d-t-s" x="427" y="162" text-anchor="middle">every sibling</text>
  <text class="d-t-s" x="427" y="178" text-anchor="middle">call site</text>

  <line class="d-line-a" x1="500" y1="103" x2="524" y2="103" marker-end="url(#aha2)"/>

  <rect class="d-box-a" x="528" y="70" width="146" height="66" rx="2"/>
  <text class="d-t-a" x="601" y="98" text-anchor="middle">unpatched</text>
  <text class="d-t-a" x="601" y="118" text-anchor="middle">siblings</text>
  <text class="d-t-s" x="601" y="162" text-anchor="middle">the odds are</text>
  <text class="d-t-s" x="601" y="178" text-anchor="middle">in your favour</text>

  <line class="d-line" x1="6" y1="210" x2="674" y2="210" stroke-dasharray="3 4"/>
  <text class="d-t-s" x="6" y="232">second output: code merged recently has had less review than code merged last year</text>
</svg>
</div>
<figcaption><b>fig. 2</b> Variant analysis, automated. The published fix is the specification for the hunt.</figcaption>
</figure>

So the first engine does not look for vulnerabilities at all. It reads the last N days of
security-relevant commits, extracts the pattern of what was fixed, and then goes hunting
for **siblings that were not patched**.

The second thing that history gives you is new attack surface. Code that landed last month
has had a month of review. Code that landed last year has had a year. Recent features are
systematically softer, and the diff tells you exactly which ones they are.

## Routing: not every target deserves the same pipeline

An early mistake was treating every repository identically. A reverse proxy and a
project-management app fail in completely different ways, and running the same checks
against both wastes time on one and misses the other.

So targets get classified first, roughly into application-layer, infrastructure-layer, and
the hybrids that most large platforms actually are. That label routes which engines run.

| target type | example shape | what runs | why |
|---|---|---|---|
| app_layer | user objects, tenants, permissions | full pipeline including taint | data flows from request to sink are the whole game |
| infra_layer | proxies, gateways, storage | diff and patterns, taint disabled | failures live in parsing and trust, not in user input reaching a sink |
| hybrid | large platforms with both | full pipeline | usually the richest, and the slowest |

Data-flow analysis is worth its cost on an application handling user objects and
permissions. On a proxy, the interesting failures live in header canonicalisation, parser
disagreements and trust boundaries, and the same analysis mostly returns noise.

## The three engines

Beyond the diff engine described above, two others run and are merged into one candidate
set.

### The pattern knowledge base

Twenty-two vulnerability classes are modelled explicitly: header canonicalisation, request
smuggling, path normalisation, trusted-IP handling, certificate verification, token
algorithm confusion, deserialisation, archive extraction, upload handling, object-level
authorisation, injection, and more.

Each entry is a description of what the mistake *looks like* in real code across the
frameworks that matter, not a regex hoping for a lucky hit. The distinction is that a
pattern encodes the surrounding context: which framework idiom puts attacker input in
scope, which helper would normally sanitise it, what the absence of that helper looks like
structurally.

Patterns are filtered by the target label and by the languages actually present, so an
infrastructure target never gets scanned for template injection in a templating engine it
does not use.

### The taint engine

Classic source-to-sink reachability: can attacker-controlled input arrive somewhere
dangerous. It understands routes and handlers across the major web frameworks, and it
follows values across function boundaries, through helpers and through object properties.

One detail that mattered more than expected: not every file announces itself with a route
decorator. Controllers, services and handlers frequently receive request objects from
somewhere else entirely. Classifying files by their *kind*, rather than waiting to find an
explicit route, is what stopped the engine from silently ignoring large parts of an
application.

The part that took the longest to get right was not finding flows. It was *grading* them.

## Scoring, not booleans

A scanner that flags everything is worse than no scanner. You stop reading its output
within a week, and then it is just a process burning CPU.

So nothing in the pipeline emits a binary verdict. Every candidate carries two independent
gradings, and both have to hold up before a human sees it.

<figure class="fig">
<div class="fig-frame">
<svg viewBox="0 0 660 380" role="img" aria-label="Funnel diagram showing candidates narrowing through flow-proof grading, validation grading and public deduplication down to a small shortlist.">
  <rect class="d-box" x="10" y="16" width="360" height="52" rx="2"/>
  <text class="d-t" x="190" y="47" text-anchor="middle">raw candidates</text>
  <line class="d-line" x1="370" y1="42" x2="398" y2="42" stroke-dasharray="2 3"/>
  <text class="d-t-s" x="406" y="46">everything the three engines emit</text>

  <rect class="d-box" x="40" y="90" width="300" height="52" rx="2"/>
  <text class="d-t" x="190" y="121" text-anchor="middle">flow-proof grading</text>
  <line class="d-line" x1="340" y1="116" x2="398" y2="116" stroke-dasharray="2 3"/>
  <text class="d-t-s" x="406" y="112">is the path actually connected,</text>
  <text class="d-t-s" x="406" y="128">or merely co-located?</text>

  <rect class="d-box" x="70" y="164" width="240" height="52" rx="2"/>
  <text class="d-t" x="190" y="195" text-anchor="middle">validation grading</text>
  <line class="d-line" x1="310" y1="190" x2="398" y2="190" stroke-dasharray="2 3"/>
  <text class="d-t-s" x="406" y="186">real guard, cosmetic guard,</text>
  <text class="d-t-s" x="406" y="202">or nothing in the way?</text>

  <rect class="d-box" x="105" y="238" width="170" height="52" rx="2"/>
  <text class="d-t" x="190" y="269" text-anchor="middle">public dedup</text>
  <line class="d-line" x1="275" y1="264" x2="398" y2="264" stroke-dasharray="2 3"/>
  <text class="d-t-s" x="406" y="260">already known to OSV,</text>
  <text class="d-t-s" x="406" y="276">GHSA or huntr?</text>

  <rect class="d-box-a" x="135" y="312" width="110" height="52" rx="2"/>
  <text class="d-t-a" x="190" y="343" text-anchor="middle">shortlist</text>
  <line class="d-line-a" x1="245" y1="338" x2="398" y2="338" stroke-dasharray="2 3"/>
  <text class="d-t-n" x="406" y="334">what a human is allowed</text>
  <text class="d-t-n" x="406" y="350">to spend time on</text>
</svg>
</div>
<figcaption><b>fig. 3</b> Every stage is designed to discard, not to accumulate. The output is deliberately small.</figcaption>
</figure>

### How well is the flow proven

Eight levels, from a direct source-to-sink call inside one function, down through helpers
and object propagation, to "the source and the sink are in the same file and nothing
connects them."

<figure class="fig">
<div class="fig-frame">
<svg viewBox="0 0 660 300" role="img" aria-label="Ladder of eight flow-proof strength levels, from direct at the top to none at the bottom, with the weakest levels marked as dropped.">
  <text class="d-t-s" x="10" y="16">flow proof</text>
  <text class="d-t-s" x="210" y="16">strength</text>
  <text class="d-t-s" x="540" y="16">outcome</text>

  <text class="d-t" x="10" y="48">direct</text>
  <rect class="d-box-a" x="210" y="34" width="300" height="18" rx="1"/>
  <text class="d-t-n" x="540" y="48">manual review</text>

  <text class="d-t" x="10" y="80">helper_sink</text>
  <rect class="d-box-a" x="210" y="66" width="262" height="18" rx="1"/>
  <text class="d-t-n" x="540" y="80">manual review</text>

  <text class="d-t" x="10" y="112">helper_return</text>
  <rect class="d-box-a" x="210" y="98" width="224" height="18" rx="1"/>
  <text class="d-t-n" x="540" y="112">manual review</text>

  <text class="d-t" x="10" y="144">object_propagation</text>
  <rect class="d-box" x="210" y="130" width="186" height="18" rx="1"/>
  <text class="d-t-s" x="540" y="144">model triage</text>

  <text class="d-t" x="10" y="176">near_sink</text>
  <rect class="d-box" x="210" y="162" width="148" height="18" rx="1"/>
  <text class="d-t-s" x="540" y="176">model triage</text>

  <text class="d-t" x="10" y="208">partial</text>
  <rect class="d-box" x="210" y="194" width="110" height="18" rx="1"/>
  <text class="d-t-s" x="540" y="208">model triage</text>

  <text class="d-t-s" x="10" y="240">source_only</text>
  <rect class="d-box-d" x="210" y="226" width="72" height="18" rx="1"/>
  <text class="d-t-s" x="540" y="240">dropped</text>

  <text class="d-t-s" x="10" y="272">none</text>
  <rect class="d-box-d" x="210" y="258" width="34" height="18" rx="1"/>
  <text class="d-t-s" x="540" y="272">dropped</text>
</svg>
</div>
<figcaption><b>fig. 4</b> Only the top of the scale earns a human. The bottom two levels are where naive scanners generate most of their noise.</figcaption>
</figure>

### How real is the validation in the way

A candidate is only interesting if nothing meaningful stands between the input and the
sink. So validation is graded too, and the grading is deliberately sceptical:

- **Strong.** Parsing to an integer, reducing to a basename, resolving and comparing
  against an allowlist. This collapses a candidate's score, because the guard genuinely
  removes the class of input that would matter.
- **Weak.** Partial checks, blocklists, checks that cover the obvious payload and nothing
  else. Lowers the score without killing it.
- **Cosmetic.** Trimming whitespace, lowercasing, checking that a value is non-empty.
  Looks protective in a diff, protects nothing. Barely moves the score.
- **None.** Full score.

This single distinction killed most of the false positives. The naive version of a taint
engine sees any function call between source and sink and assumes sanitisation. Grading the
call for what it actually does is the difference between a tool you read and a tool you
mute.

## Deduplication against public knowledge

Before anything reaches a human, surviving candidates are checked against what the world
already knows: public advisory databases and bounty platform disclosures. Candidates come
back partitioned into *novel* and *known-class*.

Rediscovering a documented issue is not research, and reading it is not worth an
afternoon. On mature targets this stage does most of the work, and doing it before triage
rather than after is the difference between a wasted day and a five-minute answer.

## Where the model sits

<div class="note">
  <span class="note-label">On the LLM question</span>
  <p>
    A language model is used, but at the end and on a shortlist, as a reviewer of ranked
    candidates, never as the thing that scans the repository. Pointed at a whole codebase
    it hallucinates plausible vulnerabilities with total confidence, and every one of them
    costs you real time to disprove. Pointed at twenty pre-scored candidates with the
    data-flow already established, it is genuinely useful: it reads the surrounding
    business logic, spots the case where reachability is technically real but practically
    gated, and drafts the exploitation path. The engineering is in what you refuse to hand it.
  </p>
</div>

## From candidate to report

The automated part ends at the shortlist. What follows is entirely manual, and it is where
most candidates die.

1. **Read the code path properly.** Not the snippet, the whole path, including the caller
   nobody thought about.
2. **Stand the thing up.** A local deployment, real configuration, realistic data.
3. **Write a proof of concept that runs.** No conditional language, no "an attacker could
   potentially". Either it reproduces or it is not a finding.
4. **Establish impact.** What does an attacker actually get, from which privilege level.
   A technically real bug with no consequence is not worth a maintainer's time.
5. **Write it up with a fix.** The report includes a concrete remediation, because the
   goal is a patch, not a trophy.

The pipeline itself is regression-tested at every stage, covering direct flows,
cross-function propagation, the validation grading, and, importantly, false-positive
rejection. Every change has to keep proving it still refuses to cry wolf.

## What it actually returns

Across the corpus so far: **82 open-source projects processed, 35 taken far enough to
produce a written report.** Self-hosted platforms, LLM tooling, ITSM and asset management,
CI/CD, developer infrastructure. Mostly projects in the range where the code is big enough
to hide things but the security team is small or nonexistent.

**53 advisories credit me across those projects. 11 are published**, and each has its own
write-up linked from [the index]({{ '/' | relative_url }}): authentication bypasses, unauthenticated
path traversal, SSRF, cross-tenant authorisation, command execution. Of the rest, 17 are still
with maintainers and 25 were closed without a public advisory, which covers everything from
"declined" to "quietly fixed".

Three more went to a vendor by email rather than through GitHub. They are accepted with a fix
scheduled, so they stay unnamed here until the release ships.

The ratio is worth stating plainly, because tooling articles usually hide it: most targets
return nothing worth reporting. That is the correct outcome. A pipeline that produces a
finding on every project is a pipeline producing fiction.

## What it does not do

Being honest about the limits is what makes the rest credible:

- It finds bug **classes**. Logic flaws that need real semantic understanding of what the
  application is *for* still walk straight past it. A broken business rule that costs money
  is invisible to a pattern engine.
- Memory-corruption and parser-crash bugs are out of scope entirely. That is fuzzing work,
  a different discipline with different tools.
- Multi-step chains where each link is individually harmless are not modelled. The engine
  reasons about single flows, not about a sequence of three requests that together defeat
  a state machine.
- On very mature, heavily audited targets, nearly everything it surfaces is already known
  or too weak to exploit. Expected, and the dedup stage exists precisely to say so quickly
  instead of wasting a day.

## Disclosure

Everything here is done against open-source code, on infrastructure I control, and
reported to maintainers before it is written about. Reports go out with a working proof of
concept and a concrete remediation, and the advisory gets published when the project is
ready, not when it would be most interesting to post.

## What comes next

The obvious gap is the one listed above: chains. Single-flow reasoning is a solved shape;
sequences are not, and that is where the interesting authorisation bugs live.

Follow-up articles will go into individual cases, starting with the ones already through
the disclosure process.
