# Research

## 1. Request

Build a minimal, clean, typography-focused **personal technical blog** hosted on GitHub Pages at `https://vnovakovskyi.github.io`. No frameworks, no CMS, no backend, no database, no JS-heavy UI. The owner wants to push the repo, enable Pages, and then add articles over time, organized with tags.

Required pages: **Home** (intro + latest 3–5 posts + social links), **Articles** (all posts with title/date/excerpt/tags), **Tags** (all tags, each with its posts), **Article** (title, date, body, tags, back link, optional prev/next). Navigation: Home, Articles, Tags. About can fold into Home.

Hard constraints: no React/Next/Astro/Vue/Tailwind or any frontend framework; no comments/login/newsletter/analytics/external services; keep v1 small and maintainable; prefer simplicity over completeness.

## 2. Context

- **Repository is greenfield.** No commits yet on `main`; only `.git/` and a `.idea/` (JetBrains) folder exist. No source files, no `docs/`.
- **Remote is already configured** to `git@github.com:vnovakovskyi/vnovakovskyi.github.io.git`.
- The repo name `vnovakovskyi.github.io` matches the owner's username, so this is a **GitHub user/organization site**. Such sites are published from the repo root and served at `https://vnovakovskyi.github.io/` (no subpath), which simplifies all internal links — relative/root-absolute paths work cleanly without a `baseurl` prefix.
- Today's date: 2026-06-19. No deadline stated.

## 3. Problem / Opportunity

The owner needs a durable, low-maintenance home for technical writing that they fully control, with near-zero operational overhead. The core tension to resolve in research: **how to get "add a new article and tags/lists update themselves" without introducing a framework.** Two failure modes to avoid:
- Over-tooling (a JS framework / SSG with a build pipeline) — violates the constraints.
- Under-tooling (hand-written HTML per page) — every new article forces manual edits to the home list, the articles list, and every relevant tag list, which does not scale and invites drift.

The opportunity is to land on the established middle path that GitHub Pages was built for.

## 4. Target Users / Stakeholders

- **Primary: the author (owner/maintainer).** Wants writing friction near zero — drop in a Markdown file, push, done. Is the only content editor and the only ops owner.
- **Readers:** engineers/technical audience arriving from GitHub, LinkedIn, or a Telegram blog. Need fast load, good readability on desktop and mobile, easy navigation and tag-based discovery.
- **Operations:** GitHub Pages is the host; no separate infra, no on-call. The author is also "ops."
- **No** security/support/product stakeholders — purely personal static content.

## 5. Current State

New project. Idea maturity is **high** — requirements, page list, tone, and constraints are already well specified; this is a design/build problem, not a discovery problem.

Existing constraints (given):
- Static hosting on GitHub Pages only.
- No frontend frameworks, no external services, no backend/DB.
- Must remain small and maintainable by one person.

Existing alternatives the author has implicitly rejected: managed CMS/blog platforms (Medium, Substack, Ghost, Hashnode) and JS SSGs (Next/Astro/Gatsby). Remaining viable approaches are evaluated in Findings.

## 6. Findings

**F1 — GitHub Pages has native Jekyll support; it is the framework-free standard path.**
GitHub Pages builds Jekyll automatically on push with **no GitHub Actions workflow, no Node, no local build step required**. Jekyll is a static-site generator (Ruby) but is invisible to the author day-to-day: you write Markdown, GitHub renders HTML. This directly satisfies "push repo → enable Pages → site is live → add articles." *Why it matters:* it's the lowest-overhead way to get auto-updating lists/tags without writing any JavaScript or maintaining a build pipeline. *Source:* GitHub Pages / Jekyll documentation (stable, long-standing behavior).

**F2 — Jekyll covers every required feature out of the box, no plugins needed.**
- Articles = Markdown files in `_posts/` named `YYYY-MM-DD-title.md` with YAML front matter (`title`, `tags`, optional `excerpt`). Adding an article = adding one file. *Satisfies requirement #4/#5 "add new articles."*
- Date comes from the filename/front matter automatically.
- Latest N posts on Home: `site.posts | limit: 5` (Liquid). *Satisfies Home "latest 3–5."*
- Full list on Articles page: loop over `site.posts`. *Satisfies Articles page.*
- Excerpts: Jekyll auto-generates `post.excerpt` (first paragraph) or you set it in front matter. *Satisfies excerpt.*
- Prev/next: `post.previous` / `post.next` are built in. *Satisfies optional prev/next — it is in fact trivial.*
- Layouts/partials via Liquid templating eliminate HTML duplication (one `default` layout, one `post` layout, shared header/nav/footer).

**F3 — Tag pages are the one nuance; they are still solvable with zero plugins.**
GitHub Pages' native build only allows a whitelist of plugins, and the common "one dedicated page per tag" generators (e.g. `jekyll-tagging`, auto-pages) are **not** on that whitelist. However, the required behavior — "list all tags, and for each tag its articles" — is fully achievable on a **single `tags` page** using built-in `site.tags` Liquid iteration, with clickable tag links pointing to anchors (`/tags/#performance`). Article cards and article pages link to those anchors. *Why it matters:* avoids any plugin and avoids the GitHub Actions route while still meeting the Tags-page requirement. The only thing not delivered is a separate URL per tag, which is not a stated requirement.

**F4 — Three implementation approaches exist; native Jekyll is the best fit.**

| Approach | Auto lists/tags | Build pipeline | Add-article effort | Verdict |
|---|---|---|---|---|
| **Native Jekyll** (recommended) | Yes (Liquid) | None (GitHub builds) | Add 1 Markdown file | Best fit |
| Plain hand-written HTML | No | None | Edit home + list + tags by hand each time | Fails maintainability |
| Jekyll/SSG via GitHub Actions | Yes (+ plugins) | Custom workflow | Add 1 file | Over-engineered for v1 |

Plain HTML is "simpler" only in tooling; it is **more** work to maintain because every new post requires touching multiple index files manually — directly at odds with the author's maintainability goal. The Actions route unlocks arbitrary plugins but adds a pipeline the author doesn't need yet.

**F5 — Typography/readability targets are well established and align with the brief.**
- Comfortable reading measure ≈ **60–75 characters per line**, i.e. a content column around **640–720px** max-width centered. *Satisfies "comfortable reading width."*
- Body font ~**18px**, line-height ~**1.6**, generous paragraph spacing. A system font stack (or one readable serif/sans) keeps it dependency-free and fast. *Satisfies typography focus + no external services (avoid even Google Fonts to honor "no external services").*
- A single small CSS file with light/responsive rules covers desktop and mobile; **no JS required** for any core feature. *Satisfies "no heavy JavaScript."*

**F6 — Optional niceties are cheap and plugin-supported on the native build.**
`jekyll-feed` (RSS), `jekyll-seo-tag` (meta tags), and `jekyll-sitemap` are on the GitHub Pages whitelist and are commonly enabled with one line each in `_config.yml`. They are out of scope per the brief (no external services / keep small) but worth noting as zero-cost future adds; an RSS feed in particular fits an engineering blog and adds no UI.

**F7 — Publishing flow is exactly the author's expected result.**
With native Jekyll: push to `main` → in repo Settings → Pages, set source to `main` branch root → GitHub builds and serves at `https://vnovakovskyi.github.io`. No `.nojekyll`, no Actions, no secrets. *Confirms expected result steps 1–3.* User sites build from the default branch by default.

## 7. Assumptions

- **A1:** The author is comfortable writing articles in **Markdown** (standard for engineers; Jekyll's native format). If they strongly prefer raw HTML or a WYSIWYG editor, the approach shifts.
- **A2:** Acceptable that tags live on a single Tags page with in-page anchors rather than one dedicated URL per tag (meets the literal requirement; no plugin/Actions needed).
- **A3:** No custom domain in v1 — the site lives at `vnovakovskyi.github.io`. (A `CNAME`/custom domain can be added later without rework.)
- **A4:** Light theme only is acceptable for v1 (dark mode not requested; can be added later in CSS).
- **A5:** The author accepts Jekyll as an invisible build dependency — it is not a "frontend framework" in the sense the constraints exclude (no React/Vue/etc.), and requires no local toolchain.
- **A6:** Repo will be public (required for free GitHub Pages on personal accounts).
- **A7:** Content/personal details (real name, bio text, GitHub/LinkedIn/Telegram URLs) will be supplied by the author during build.

## 8. Risks

- **R1 — Jekyll classified as a "framework" by the author.** The brief bans frontend frameworks; Jekyll is a backend-only static generator with no client runtime, but if the author wants *zero* generator, fall back to plain HTML and accept manual index maintenance. *Mitigation:* confirm this framing in planning (see Open Questions).
- **R2 — Local preview friction.** Previewing Jekyll locally needs Ruby + `bundler` + `jekyll`. *Mitigation:* not required to publish — the author can preview by pushing to a branch, or skip local preview entirely for v1. Low impact.
- **R3 — Tag UX limitation.** No per-tag permalink URLs without a plugin/Actions. *Mitigation:* anchor-based single Tags page is sufficient for the stated requirement; revisit only if per-tag URLs become desired.
- **R4 — Markdown lock-in to author's writing habit.** Low risk for an engineering audience; Markdown is the norm.
- **R5 — Scope creep.** Comments, search, analytics, dark mode, RSS are all tempting. *Mitigation:* explicitly deferred per the brief; keep v1 to the four page types.
- **R6 — Date/timezone quirks in Jekyll post ordering.** Minor; posts order by front-matter date. *Mitigation:* always include a date in front matter or filename.
- **Cost/operational risk:** effectively none — free hosting, no services, no secrets to leak (no analytics/login/external integrations by design).

## 9. Open Questions

1. **Generator vs none:** Is the author OK with **native Jekyll** (recommended, framework-free in the client sense) as the build mechanism, or do they want truly hand-written static HTML with manual list maintenance? *(This is the one decision that changes the architecture.)*
2. **Tags:** Is a **single Tags page with anchors** acceptable, or are dedicated per-tag URLs required (would push toward the GitHub Actions build)?
3. **Home as About:** Confirm folding About into Home (recommended for simplicity) vs a separate About page.
4. **Identity/content:** What name, one-paragraph bio, and exact **GitHub / LinkedIn / Telegram** URLs should appear? (Content, not a blocker for architecture.)
5. **Aesthetic preference:** serif vs sans body font; light-only vs include dark mode later. (Default: system sans, light-only.)
6. **RSS feed:** include `jekyll-feed` (one line, zero UI) or strictly omit per "no extras"? (Default: omit in v1.)

## 10. Recommendation

**Continue to `plan`.** The problem is well understood and the solution space is narrow. The recommended approach is **native Jekyll on GitHub Pages with no plugins and no GitHub Actions** — it uniquely satisfies all functional requirements (auto-updating lists, tags, dates, excerpts, prev/next) while honoring every constraint (no frontend framework, no backend, no external services, minimal and maintainable). A full architecture or data-model phase is **not** warranted for a project this small; the "data model" is just post front matter and is best defined inline in the plan.

Before planning is finalized, resolve **Open Question #1** (Jekyll acceptable?) and **#2** (single Tags page acceptable?), since those two answers determine the build mechanism. Questions #3–#6 can be defaulted as noted and confirmed during the build.

## 11. Next Action

Run the **`plan`** phase to produce the MVP plan. Suggested scope to plan around:
- Native Jekyll site at repo root: `_config.yml`, `_layouts/{default,post}.html`, `index.html` (Home), `articles.html`, `tags.html`, a single `assets/style.css`, `_posts/` with one sample article, and a `.gitignore`.
- Confirm Open Questions #1 and #2 with the author first (recommend: yes to Jekyll, yes to single Tags page).
- Publishing steps: push `main`, Settings → Pages → source = `main`/root, verify at `https://vnovakovskyi.github.io`.
