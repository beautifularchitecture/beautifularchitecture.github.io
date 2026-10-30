# AGENTS.md — beautifularchitecture.github.io

Hugo static site (hugoplate theme, TailwindCSS v4, Hugo modules). Deployed to GitHub Pages via `.github/workflows/deploy.yml`. **No `opencode.json`** — instructions are served by this file plus `.github/copilot-instructions.md`.

## Commands

| Command                  | Purpose                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `npm run dev`            | Hugo dev server (localhost:1313, live reload; **no drafts** — use `hugo server -D` for drafts)                                  |
| `npm run build`          | Production build (gc, minify, template metrics, `--forceSyncStatic`)                                                            |
| `npm run preview`        | Dev server, production env, template metrics                                                                                    |
| `npm run format`         | Prettier only (with `prettier-plugin-go-template` + `prettier-plugin-tailwindcss`) — **no linter, no typecheck, no test suite** |
| `npm run update-modules` | Update all Hugo module deps                                                                                                     |

## CI / deploy

Push to `main` → GitHub Actions (Node 24, Hugo latest) → GitHub Pages. CI builds with `hugo --gc --minify --baseURL "$URL"` (not `npm run build`) — GitHub Pages injects the URL. CI uses `npm ci`.

**Tracked despite `.gitignore`** (CI needs them): `package-lock.json` and `hugo_stats.json` — do not delete. `hugo_stats.json` churns whenever content tags/classes change; commits routinely include it alongside content edits — that's expected, not drift.

## Writing rules

- **Never write ambiguous or unclear text.** Every sentence must have one unambiguous meaning. No hedging ("usually", "most of the time", "it depends"), no vague pronouns ("this", "that", "it") whose referent is unclear, no implied-but-unstated steps. State exactly what applies.
- **Verify every fact before writing it.** Do not state anything you have not confirmed by reading the repo, running the command or checking the linked source. Verify tool flags, file paths, versions and config values against the actual source (`package.json`, `hugo.toml`, `config/_default/`, workflow files, the vendored theme) rather than from memory.
- **Cite the source for external facts.** Whenever you reference documentation, a spec, a CVE, a version number or a claim about how a third-party tool behaves, include the URL. For repo-specific claims, reference the exact file path (and line number when useful).
- **If you cannot verify it, say so.** Explicitly flag unverified claims as unverified instead of guessing. Never invent file paths, flags, config keys, dates or URLs.

## Content

- **Frontmatter is TOML** (`+++`), never YAML.
- Blog posts: `content/english/blog/{category}/{slug}.md`.
- **Do not manually number blog headings solely for the section hierarchy.** Blog H2 headings are numbered automatically as `1.`, `2.` and blog H3 headings as `1.1`, `1.2`, with H3 numbering restarting under each H2. Preserve meaningful labels such as `Part 1`, `Phase 1`, `Step 1` and `Pattern 1` in the heading text.
- 10 categories: `ai/`, `business-ideas/`, `containerization/`, `languages/`, `legacy/`, `misc/`, `roadmap/`, `security/`, `soft-skills/`, `system_design/`.
- `summaryLength = 10` — excerpts truncate at 10 words.
- No Oxford comma (see `.github/instructions/blog.instructions.md`).
- Goldmark renders raw HTML in markdown (`unsafe = true`).
- Code highlighting: `guessSyntax = true`, `solarized-light` style.
- **Blog post dates must always be yesterday's date** (not today's). All dates in a batch must be unique and at least 5 seconds apart — duplicate dates cause Hugo build breakages (two pages with the same timestamp collide). Use the same `+10:00` timezone offset as existing posts.
- **Do not create `_index.md` in sub-section directories** (e.g. `blog/ai/_index.md`) — it causes Hugo to treat the sub-section as separate from the parent `blog/`, excluding those posts from the main `/blog/` listing. Only the top-level `blog/_index.md` should exist.

## Architecture

- **Theme**: `github.com/zeon-studio/hugoplate` is **vendored** in `themes/hugoplate/` and committed to the repo (115 files). `hugo.toml` sets `theme = "hugoplate"`, and the build fails if `themes/` is removed — the local copy is the active source, not an auto-downloaded module. If you edit theme templates/styles, edit `themes/hugoplate/`, not the `config/_default/module.toml` import. Helper modules (search, mermaid, icons, etc.) are declared in `config/_default/module.toml`.
- **Asset pipeline**: TailwindCSS v4 via `@tailwindcss/cli` (no `tailwind.config.js`, no PostCSS). Entrypoint: `assets/css/custom.css`.
- **Custom plugins**: `tailwind-plugin/` (`tw-bs-grid.js`, `tw-theme.js`).
- **Mermaid diagrams**: supported via `github.com/hugomods/mermaid` Hugo module (CDN).
- **Custom layouts**: only `layouts/shortcodes/` (`iframe.html`, `include.html`) — everything else inherited from the theme.
- **Required**: Hugo extended >= 0.151.0. Go and Node.js needed locally (CI uses Node 24, no `.nvmrc`).
- **Build output**: `public/` — generated, not committed.
- **Quirk**: `hugo_stats.json` mounted with `disableWatch = true` — CSS rebuilds won't trigger on stat changes alone.

## See also

- `.github/copilot-instructions.md` — writing style, anti-hallucination rules
- `.github/instructions/blog.instructions.md` — blog editorial rules
- `.github/commit-instructions.md` — commit message format
- `.github/agents/` — Codex agent definitions (not OpenCode)
