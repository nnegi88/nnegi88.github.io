# nnegi88.github.io

Personal site. Plain Jekyll on GitHub Pages — markdown in, HTML out, no build tooling to maintain.

## Publish a post

Drafts live in `_drafts/` and are **not** published. To publish:

```sh
git mv _drafts/some-post.md _posts/2026-08-12-some-post.md   # date prefix is required
git commit -am "publish: some post"
git push
```

Live at https://nnegi88.github.io in about a minute.

## Write a new post

Create `_drafts/my-title.md`:

```markdown
---
layout: post
title: "The title"
subtitle: "One line that sets expectation."
---

Body in markdown.
```

## Preview locally (optional)

```sh
bundle exec jekyll serve --drafts
```

Not required — pushing is the fast path.

## Custom domain later

Buy a domain, add a `CNAME` file containing it, point DNS at GitHub. Every existing
link keeps working.
