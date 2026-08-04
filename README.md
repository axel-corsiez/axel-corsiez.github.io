# axel-corsiez.github.io

Personal research site: offensive security notes and vulnerability research write-ups.
Published at <https://axel-corsiez.github.io/>.

Static Jekyll site, no theme gem, no plugins, no external assets.

## Adding an article

Drop a Markdown file in `_posts/` named `YYYY-MM-DD-slug.md`:

```markdown
---
layout: post
title: "Title of the article"
summary: "One sentence shown on the home page."
reading_time: "6 min"
tags: [tag-one, tag-two]
---

Body in Markdown.
```

It appears on the home page automatically, newest first.

## Local preview

```bash
bundle exec jekyll serve
```

## Disclosure policy

Nothing unpatched is published here. Findings go to maintainers first; write-ups follow
the fix.
