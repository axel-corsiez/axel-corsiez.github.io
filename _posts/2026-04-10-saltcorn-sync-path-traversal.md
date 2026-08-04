---
layout: post
title: "CVE-2026-40163: the sanitiser was in the same file, two functions up"
summary: "Saltcorn had already been patched for path traversal in its sync directory once. The helper written for that fix was never applied to the two neighbouring endpoints, both unauthenticated."
reading_time: "7 min"
tags: [cve, path-traversal, nodejs, variant-analysis]
lang: en
ref: saltcorn-sync-path-traversal
cve: CVE-2026-40163
---

**CVE-2026-40163** · [GHSA-32pv-mpqg-h292](https://github.com/advisories/GHSA-32pv-mpqg-h292) ·
Saltcorn (`@saltcorn/server`) · CWE-22 · High, CVSS 8.2 · published 2026-04-10 · fixed in
1.4.5, 1.5.5 and 1.6.0-beta.4

This is the case I point at when I explain why I read commit history before I read code.

Saltcorn had already received a path traversal advisory in its mobile sync feature
(GHSA-43f3-h63w-p6f6). The fix introduced a validation helper, `File.normalise_in_base()`,
and applied it to the endpoint that had been reported.

The helper was correct. It was applied to one endpoint. Two others in the **same file** kept
building paths by hand, and neither required authentication.

## Finding 1: arbitrary file write

`POST /sync/offline_changes` takes a timestamp from the request body and uses it to name a
directory:

```javascript
const syncDirName = `${newSyncTimestamp}_${req.user?.email || "public"}`;
const syncDir = path.join(
    rootFolder.location, "mobile_app", "sync", syncDirName
);
await fs.mkdir(syncDir, { recursive: true });
await fs.writeFile(
    path.join(syncDir, "changes.json"),
    JSON.stringify(changes)
);
```

`path.join()` is not a security boundary. It resolves `..` as part of normal operation, which
is exactly what it is for. Set `newSyncTimestamp` to `../../../../tmp/evil` and the path lands
wherever you aimed it.

Note `req.user?.email || "public"` on the first line. The optional chaining and the fallback
are the code telling you, in passing, that it expects to run without a user. There is no auth
middleware on the route.

So: unauthenticated directory creation anywhere the process can write, plus a file called
`changes.json` whose content is attacker-supplied JSON.

## Finding 2: arbitrary directory read

`GET /sync/upload_finished` takes `dir_name` from the query string and joins it the same way.
Same missing helper, same missing authentication, opposite direction: list the contents of any
directory on the host, and read specific JSON files from it.

## Variant analysis, which is most of the job

None of this required deep understanding of Saltcorn. It required noticing that a fix had
happened and asking the follow-up question the fix does not answer: *where else does this
pattern live?*

A security patch publishes three things at once. It closes one hole, it names the sanitiser,
and it tells you the maintainer now considers this class relevant to this file. The third is
the useful one. If somebody had to write `normalise_in_base()` to make one endpoint safe, then
every path built without it in that neighbourhood is a candidate, and the neighbourhood is
usually the same file.

Here the distance between the patched endpoint and the unpatched ones was about sixty lines.

This is the engine I built the pipeline around and wrote up in
[the article on hunting at scale]({{ '/2026/08/hunting-cves-at-scale/' | relative_url }}):
read recent security commits, extract the shape of what was fixed, then hunt the siblings that
were not. Saltcorn is the clean illustration, because the sanitiser was not merely available,
it was *right there*.

## Impact and fix

Unauthenticated arbitrary file write with partial content control, and unauthenticated
directory listing plus JSON read. Writing attacker-controlled JSON to a chosen path on a
platform that loads JSON configuration is a comfortable position to build from.

Fixed across three release lines: 1.4.5, 1.5.5, 1.6.0-beta.4. The fix applies the existing
helper to both endpoints.

## Timeline

- Reported to the maintainers
- Fixed in 1.4.5 / 1.5.5 / 1.6.0-beta.4
- Advisory [GHSA-32pv-mpqg-h292](https://github.com/advisories/GHSA-32pv-mpqg-h292) published on 2026-04-10
- CVE-2026-40163 assigned
