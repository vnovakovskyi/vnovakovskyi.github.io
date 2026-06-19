# vnovakovskyi.github.io

My personal technical blog — a minimal, typography-focused static site built
with [Jekyll](https://jekyllrb.com/) and hosted on GitHub Pages at
<https://vnovakovskyi.github.io>.

No frameworks, no JavaScript, no external services. Just Markdown and HTML.

## Add a new article

Create a Markdown file in `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
title: "Your title"
date: 2026-06-19
tags: [backend, systems]
excerpt: "One-sentence summary shown on the Articles page."
---

Your intro paragraph.

<!--more-->

The rest of the article...
```

Commit and push to `main`. The Home list, Articles page, and Tags page update
automatically — no other files to edit.

- `excerpt` is optional; if omitted, the first paragraph (up to `<!--more-->`)
  is used.
- `tags` drive the Tags page. Tags are linked from article cards and article
  pages to `tags.html#<tag>`.

## Enable GitHub Pages

1. Push this repo to `main`.
2. In the repo: **Settings → Pages**.
3. Under **Build and deployment**, set **Source = Deploy from a branch**,
   **Branch = `main` / `(root)`**, and save.
4. Wait for the build, then open <https://vnovakovskyi.github.io>.

## Local preview (optional)

Requires Ruby ≥ 2.7 and Bundler:

```sh
bundle install
bundle exec jekyll serve
# open http://localhost:4000
```

Local preview is not required to publish — GitHub Pages builds the site on push.

## Layout

```
_config.yml        Site config + social links
_layouts/          default.html (shell) and post.html (article)
_posts/            Articles, as Markdown
assets/style.css   All styles
index.html         Home
articles.html      All articles
tags.html          Tags
docs/              Research & plan notes
```
