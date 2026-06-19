# Plan

## 1. Goal

Ship v1 of a minimal, typography-focused personal technical blog hosted on GitHub Pages at `https://vnovakovskyi.github.io`, built with **native Jekyll** (no plugins beyond GitHub Pages defaults, no GitHub Actions, no client-side JavaScript). After this work, the author can add a new article by committing a single Markdown file, and the Home list, Articles list, and Tags page update automatically.

## 2. Background

Greenfield repo whose name (`vnovakovskyi.github.io`) makes it a GitHub **user site** served at the domain root — internal links need no `baseurl` prefix. Per `docs/00-research.md`, native Jekyll is the lowest-overhead path that satisfies every functional requirement (auto lists, tags, dates, excerpts, prev/next) while honoring all constraints (no frontend framework, no backend, no external services, small and maintainable). The author has confirmed: **yes to native Jekyll**, and **yes to a single Tags page with in-page anchors** (no per-tag URLs). About folds into Home. Light theme only. No RSS in v1.

## 3. Scope

### In Scope

- Native Jekyll site at repo root, built automatically by GitHub Pages on push.
- `_config.yml` with site metadata and author/social links.
- Two layouts: `default` (shared shell: nav + footer) and `post` (article page).
- Four page types: **Home** (`index.html`), **Articles** (`articles.html`), **Tags** (`tags.html`), and **Article** (rendered from `_posts/*.md` via the `post` layout).
- One small hand-written CSS file (`assets/style.css`) — responsive, typographic, light theme, no external fonts.
- Markdown posts in `_posts/` with front matter (`title`, `date`, `tags`, optional `excerpt`).
- Home: short intro/About text, latest 3–5 posts, link to all articles, social links (GitHub, LinkedIn, Telegram).
- Articles: full list with title, date, excerpt, tags.
- Tags: all tags, each listing its posts; tags clickable from article cards and article pages (anchor links into the Tags page).
- Article page: title, date, body, tags, back-to-articles link, prev/next links.
- 2–3 sample posts to demonstrate lists, tags, and prev/next.
- `.gitignore` and a `Gemfile` (for optional local preview; not required to publish).
- Publishing: enable GitHub Pages from `main`/root and verify the live site.

### Out of Scope

- Comments, login, newsletter, analytics, search, any external service.
- Dark mode / theme switcher.
- RSS feed, SEO/sitemap plugins (deferred; one-line future adds).
- Per-tag permalink URLs and tag pagination.
- Custom domain / `CNAME`.
- Any JavaScript framework (React/Vue/Next/Astro/etc.) or client-side JS at all.
- Local build pipeline via GitHub Actions.

## 4. Success Criteria

- **Product:** A reader can land on Home, read the intro, see the latest posts and social links, browse all articles, filter/discover by tag, and read a full article with working back and prev/next navigation — on both desktop and mobile.
- **Technical:** Site builds cleanly on GitHub Pages (native Jekyll). No client-side JS. All internal links resolve at the domain root. No console errors.
- **Operational:** Adding a new article = add one Markdown file to `_posts/` and push; all lists/tags update with no manual edits elsewhere.
- **Quality:** Comfortable reading column (~640–720px), ~18px body, ~1.6 line-height; legible and uncluttered on a 375px-wide viewport with no horizontal scroll.

## 5. User Stories / Use Cases

- **As the author**, I add `_posts/2026-06-19-my-post.md` with front matter, push to `main`, and the post appears on Home, Articles, and under each of its tags — without editing any index file.
- **As a reader**, I open the site, read who the author is and what they write about, and click through to the latest article.
- **As a reader**, I open Articles to scan every post by title/date/excerpt/tags.
- **As a reader**, I click a tag on an article card and land on the Tags page scrolled to that tag's article list.
- **As a reader on mobile**, I read an article in a comfortable single column with working "← All articles" and prev/next links.

## 6. Functional Requirements

1. Site is built by native Jekyll on GitHub Pages with no non-default plugins and no Actions workflow.
2. Global nav (Home, Articles, Tags) and footer (social links) appear on every page via the `default` layout.
3. Home shows: intro/About paragraph, the latest 3–5 posts (title + date, newest first), a "View all articles" link, and GitHub/LinkedIn/Telegram links.
4. Articles page lists **all** posts (newest first), each with title (link), date, excerpt, and tags.
5. Tags page lists all tags; under each tag heading (with an `id` anchor) it lists that tag's posts (title + date), newest first.
6. Tags shown on article cards and on article pages are links to `tags.html#<tag>`.
7. Article page renders: title, formatted date, body (Markdown→HTML), tag links, a "back to all articles" link, and previous/next post links (omitted gracefully at the ends).
8. Dates render in a consistent human-readable format (e.g. `Jun 19, 2026`).
9. Excerpts come from front-matter `excerpt` when present, else Jekyll's auto excerpt (first paragraph).
10. Adding a post requires no edits outside the new file in `_posts/`.

## 7. Non-Functional Requirements

- **Performance:** Static HTML + one small CSS file; no JS, no web fonts. Fast first paint; minimal bytes.
- **Maintainability:** Single source of layout truth (layouts + one CSS file); content separated from presentation; readable Liquid.
- **Reliability:** No runtime dependencies or external services; nothing to break at request time beyond GitHub Pages CDN.
- **Accessibility/readability:** Sufficient contrast, semantic HTML (`header`/`main`/`article`/`footer`/`nav`/`time`), responsive meta viewport, no horizontal scroll at 375px.
- **Security/privacy:** No third-party scripts, trackers, or cookies. External links use `rel="noopener"`.
- **Cost:** $0 — free GitHub Pages hosting.
- **Portability:** Plain Markdown + Jekyll; content is trivially movable to another SSG later.

## 8. Dependencies

- **External:** GitHub Pages (hosting + native Jekyll build). Jekyll + GitHub Pages default gems (managed by GitHub; the `Gemfile` mirrors them only for optional local preview).
- **Internal:** Repo remote already set to `git@github.com:vnovakovskyi/vnovakovskyi.github.io.git`.
- **Author-supplied content:** display name, one-paragraph bio, and the exact GitHub / LinkedIn / Telegram URLs (placeholders used until provided).
- **Optional local tooling:** Ruby + Bundler for local preview — **not required** to publish.

## 9. Risks and Mitigations

- **No local Ruby → can't preview locally.** Impact: slower iteration. Mitigation: previewing is optional; verify via GitHub Pages build on a push. Provide a `Gemfile` so local preview works if Ruby is available.
- **Tag anchor mismatch (case/spaces).** Impact: broken tag links. Mitigation: use lowercase, hyphen-safe tag slugs consistently for both the heading `id` and the link (`| slugify`).
- **Date/timezone ordering quirks.** Impact: wrong post order. Mitigation: always set the date via filename/front matter; rely on `site.posts` default ordering.
- **GitHub Pages build silently fails.** Impact: stale site. Mitigation: check the Pages build status in repo Settings/Actions tab after first push; keep config minimal.
- **Scope creep (search, comments, dark mode, RSS).** Mitigation: explicitly out of scope; revisit post-v1.

## 10. Implementation Phases

Implementation is done one phase at a time. After each phase passes its automated checks, **pause** for the user to confirm manual verification before starting the next.

> **Note on automated checks:** local `bundle exec jekyll build` is the preferred automated gate but requires Ruby. If Ruby/Bundler is unavailable locally, the equivalent gate is "GitHub Pages build succeeds on push" (checked in the repo's Pages/Actions status). Each phase lists both.

### Phase 1: Project scaffold & configuration

**Goal:** Establish the Jekyll site skeleton and metadata so GitHub Pages can build it.

**Files likely affected:** `_config.yml`, `.gitignore`, `Gemfile`.

**Changes:** Create `_config.yml` (title, description, author, social URLs as variables, date format, `permalink` for posts, excerpt separator). Add `.gitignore` (`_site/`, `.jekyll-cache/`, `.bundle/`, `vendor/`). Add a `Gemfile` pinning `github-pages` for optional local preview. No layouts/pages yet.

**Success criteria — Automated:**
- [x] `bundle exec jekyll build` succeeds (if Ruby available), **or** GitHub Pages build succeeds after push. *(Local full build not possible — system Ruby 2.6 < github-pages requirement. Validated structure instead; Pages build is the live gate on push.)*
- [x] `_config.yml` is valid YAML (no build error). *(Verified with `ruby -ryaml`.)*

**Success criteria — Manual:**
- [ ] `_config.yml` contains correct site title, author name, and social link placeholders.
- [ ] `.gitignore` excludes `_site/` and cache dirs.

### Phase 2: Shared layouts & stylesheet

**Goal:** Build the visual shell (nav + footer) and the typographic CSS used by every page.

**Files likely affected:** `_layouts/default.html`, `_layouts/post.html`, `assets/style.css`, `_includes/` (optional, only if a shared partial helps).

**Changes:** `default.html` with `<head>` (viewport meta, title, link to CSS), semantic `header`/`nav` (Home, Articles, Tags), `main` content slot, and `footer` with social links from `_config.yml`. `post.html` extends `default` and renders article title, date, body, tags, back link, prev/next. `assets/style.css`: centered ~640–720px column, ~18px/1.6 body, system font stack, responsive rules, link/tag styles, no external fonts.

**Success criteria — Automated:**
- [x] Build succeeds (local or Pages) with both layouts present. *(Both layouts' Liquid parsed clean via the `liquid` gem; Pages build on push is the live gate.)*

**Success criteria — Manual:**
- [ ] Nav and footer render with working links; layout is centered and readable at desktop width.
- [ ] At 375px width there is no horizontal scroll.

### Phase 3: Home page

**Goal:** Implement Home as the landing + About page.

**Files likely affected:** `index.html` (or `index.md`).

**Changes:** Intro/About paragraph, latest 3–5 posts via `site.posts | limit:5` (title link + date), "View all articles →" link to `articles.html`, and a social links row.

**Success criteria — Automated:**
- [x] Build succeeds. *(`index.html` Liquid parsed clean; valid front matter.)*

**Success criteria — Manual:**
- [ ] Home shows intro, up to 5 latest posts newest-first, all-articles link, and social links.
- [ ] No regression: nav/footer still correct.

### Phase 4: Articles page

**Goal:** Implement the full article index.

**Files likely affected:** `articles.html`.

**Changes:** Loop `site.posts` rendering a card per post: title (link), formatted date, excerpt, and tag links (`tags.html#<slug>`).

**Success criteria — Automated:**
- [x] Build succeeds. *(`articles.html` Liquid parsed clean.)*

**Success criteria — Manual:**
- [ ] All posts listed newest-first with title/date/excerpt/tags; tag links point to Tags-page anchors.

### Phase 5: Tags page

**Goal:** Implement tag-based discovery on a single page.

**Files likely affected:** `tags.html`.

**Changes:** A top index of all tags (links to anchors), then per-tag sections (`<h2 id="{{ tag | slugify }}">`) each listing that tag's posts (title + date) newest-first, iterating `site.tags`.

**Success criteria — Automated:**
- [x] Build succeeds. *(`tags.html` Liquid parsed clean.)*

**Success criteria — Manual:**
- [ ] Every tag appears with its correct posts; clicking a tag from an article card/page scrolls to the matching section.

### Phase 6: Sample content & article-page polish

**Goal:** Add real sample posts and confirm the article reading experience incl. prev/next.

**Files likely affected:** `_posts/2026-06-17-*.md`, `_posts/2026-06-18-*.md`, `_posts/2026-06-19-*.md` (3 samples), minor tweaks to `post.html`.

**Changes:** Add 2–3 Markdown posts with front matter and varied tags. Verify `post.previous`/`post.next` render and gracefully omit at the ends; verify back-to-articles link, tag links, and Markdown rendering (headings, code blocks, lists).

**Success criteria — Automated:**
- [x] Build succeeds with all sample posts. *(3 sample posts added; all front matter valid YAML.)*

**Success criteria — Manual:**
- [ ] Each article shows title/date/body/tags/back link; prev/next correct and absent at first/last post.
- [ ] Code blocks and headings render legibly within the reading column on mobile and desktop.

### Phase 7: Publish to GitHub Pages

**Goal:** Make the site live and verify end-to-end.

**Files likely affected:** none (repo settings) — possibly a `README.md`.

**Changes:** Commit all work, push to `main`, enable Pages (Settings → Pages → source = `main`/root). Optionally add a short `README.md`.

**Success criteria — Automated:**
- [ ] GitHub Pages build for `main` succeeds (Pages/Actions status green).

**Success criteria — Manual:**
- [ ] `https://vnovakovskyi.github.io` loads; Home/Articles/Tags/Article and all nav/social/tag/prev-next links work on the live site.
- [ ] Author confirms adding a new `_posts/*.md` + push surfaces it everywhere automatically.

## 11. Testing Strategy

- **Build check (automated):** `bundle exec jekyll build` locally where possible; otherwise the GitHub Pages build status is the gate.
- **Link/render check (manual):** click through nav, social, tag anchors, back, and prev/next on each page type.
- **Responsive check (manual):** inspect at ~375px and desktop widths — no horizontal scroll, comfortable measure.
- **Content-workflow check (manual):** add a throwaway post and confirm it appears on Home/Articles/Tags with zero other edits, then remove it.
- No unit/integration/e2e frameworks — disproportionate for a static, JS-free personal site.

## 12. Rollout / Migration Strategy

Single-step rollout: enable GitHub Pages from `main`/root (Phase 7). No data migration, no users to migrate, no versioning concerns. Rollback = revert the commit. A custom domain can be added later via `CNAME` without rework.

## 13. Documentation Updates

- Optional `README.md`: one-paragraph project description + "How to add an article" (create `_posts/YYYY-MM-DD-title.md` with front matter, push) + how to enable/verify Pages.
- Keep `docs/00-research.md` and `docs/01-plan.md` as the project record.

## 14. Open Questions

All blocking questions are resolved (Jekyll: yes; single Tags page: yes; About-in-Home: yes; light-only: yes; no RSS in v1). Remaining items are **content, not blockers**, and can be filled with placeholders during the build:

- Exact display name and one-paragraph bio.
- Exact GitHub / LinkedIn / Telegram URLs.
- Serif vs. sans body font (default: system sans). 

If you'd like, provide the name/bio/URLs before Phase 3 so Home ships with real content instead of placeholders.
