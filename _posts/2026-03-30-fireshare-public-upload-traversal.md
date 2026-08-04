---
layout: post
title: "CVE-2026-34745: the patched endpoint had an unauthenticated twin"
summary: "Fireshare had already been fixed for path traversal in its chunked upload. The same file held a public version of that endpoint, and the fix had not been applied to it."
reading_time: "5 min"
tags: [cve, path-traversal, variant-analysis, python]
lang: en
ref: fireshare-public-upload-traversal
cve: CVE-2026-34745
---

**CVE-2026-34745** · [GHSA-fvvp-rj8g-c7gc](https://github.com/ShaneIsrael/fireshare/security/advisories/GHSA-fvvp-rj8g-c7gc) ·
Fireshare · CWE-22 · Critical, CVSS 9.1 · published 2026-03-30 · fixed in 1.5.3

Fireshare is a self-hosted media sharing app. It had already received a path traversal
advisory, CVE-2026-33645, on its chunked upload endpoint. That was reported, fixed, published.

The endpoint that got fixed was `POST /api/uploadChunked`, which requires authentication.
Roughly a thousand lines below it, in the same file, sits `POST /api/uploadChunked/public`,
which does not. The fix stopped at the first one.

## The same bug, one privilege level lower

```python
@api.route('/api/uploadChunked/public', methods=['POST'])
def public_upload_videoChunked():
    checkSum = request.form.get('checkSum')          # no sanitisation
    tempPath = os.path.join(upload_directory, f"{checkSum}.{filetype}")
    with open(tempPath, 'ab') as f:
        f.write(blob.read())                          # attacker-controlled content
```

`checkSum` is supposed to be a hash. It is used to name the temporary file, and it goes
straight into `os.path.join()`, which resolves `..` because that is its job. Point it out of
the upload directory and the write lands wherever you aimed it, with content you supply.

The result is strictly worse than the original CVE. That one needed a valid account: `PR:L`.
This one needs nothing: `PR:N`. Same class, same file, higher impact, because the twin
endpoint is the public one.

## Why the twin gets missed

A security fix arrives as a diff, and a diff draws your eye to the lines it touches. The
question that does not get asked in that moment is: *does this function have relatives?*

Public and private variants of the same handler are one of the most reliable places to find
them. They exist because someone needed the feature without a login, so they copied the
authenticated version and removed the auth decorator. From that moment the two drift
independently, and a fix applied to one has no reason to reach the other.

The tell here was mechanical. The upload directory is built the same way in both functions,
the parameter has the same name, and only one of them had gained a sanitiser. You do not need
to understand Fireshare to see that.

This is the pattern my pipeline is built around, described in
[hunting CVEs at scale]({{ '/2026/08/hunting-cves-at-scale/' | relative_url }}): when a project
publishes a fix, it publishes the shape of its own weak code. Read the fix, then hunt the
siblings.

## Impact and fix

Unauthenticated arbitrary file write with attacker-controlled content, anywhere the process
can write. On a media server that usually runs with generous filesystem permissions, that is
the end of the conversation.

Fixed in 1.5.3 by applying the existing sanitisation to the public endpoint.

## Timeline

- CVE-2026-33645 fixed on the authenticated endpoint by the maintainer
- Public twin reported
- Fixed in 1.5.3
- Advisory published 2026-03-30, CVE-2026-34745 assigned
