# AGENTS.md — beautifularchitecture.github.io

Hugo static site (vendored hugoplate theme, TailwindCSS v4 compiled by Hugo itself). Deployed to GitHub Pages via `.github/workflows/deploy.yml`. **No `opencode.json`** — instructions are served by this file plus `.github/copilot-instructions.md`. Those two files duplicate the same conventions, so update both when a convention changes.

## Commands

| Command                  | Purpose                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `npm run dev`            | Hugo dev server (localhost:1313, live reload; **drafts excluded** — use `hugo server -D`)                            |
| `npm run build`          | Production build (gc, minify, template metrics, `--forceSyncStatic`). Takes ~4 s and is the only verification step.  |
| `npm run preview`        | Dev server, production env, template metrics                                                                         |
| `npm run format`         | `prettier -w .` over the **whole repo** (no `.prettierignore`) — prefer `npx prettier -w <path>` for targeted edits   |
| `npm run dev:lan`        | LAN dev server, but the IP is hardcoded in `package.json`; `README.md` documents the `hostname -I` alternative      |

There is **no linter, typecheck or test suite** — validate with `npm run build`.

**Broken scripts:** `update-modules`, `remove-darkmode`, `project-setup`, `theme-setup` and `update-theme` call `./scripts/*.js`, but no root `scripts/` directory exists. The theme's own copies live in `themes/hugoplate/scripts/`. Do not "fix" these by running them; run `hugo mod get -u ./... && hugo mod tidy` directly if module updates are needed.

## CI / deploy

Push to `main` (or manual dispatch) → GitHub Actions → GitHub Pages. The workflow installs Node 24, resolves the latest Hugo extended release from the GitHub API, runs `npm ci`, then builds with `hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"` (not `npm run build`).

**Tracked despite `.gitignore`** (CI needs them): `package-lock.json` and `hugo_stats.json` — do not delete. `npm run build` rewrites `hugo_stats.json`, which is normal churn: commit it alongside content edits and never hand-edit it.

## Writing rules

- **Never write ambiguous or unclear text.** Every sentence must have one unambiguous meaning. No hedging ("usually", "most of the time", "it depends"), no vague pronouns ("this", "that", "it") whose referent is unclear, no implied-but-unstated steps. State exactly what applies.
- **Verify every fact before writing it.** Do not state anything you have not confirmed by reading the repo, running the command or checking the linked source. Verify tool flags, file paths, versions and config values against the actual source (`package.json`, `hugo.toml`, `config/_default/`, workflow files, the vendored theme) rather than from memory.
- **Cite the source for external facts.** Whenever you reference documentation, a spec, a CVE, a version number or a claim about how a third-party tool behaves, include the URL. For repo-specific claims, reference the exact file path (and line number when useful).
- **If you cannot verify it, say so.** Explicitly flag unverified claims as unverified instead of guessing. Never invent file paths, flags, config keys, dates or URLs.

## Content

- **Frontmatter is TOML** (`+++`), never YAML. `archetypes/default.md` sets `draft = true`, so `hugo new` output is invisible in `npm run dev` until the draft flag is cleared.
- Blog posts: `content/english/blog/{category}/{slug}.md`; standalone pages: `content/english/pages/`.
- **Do not manually number blog headings solely for the section hierarchy.** CSS counters in `assets/css/custom.css` number blog H2 headings as `1.`, `2.` and H3 headings as `1.1`, `1.2`, with H3 numbering restarting under each H2. Preserve meaningful labels such as `Part 1`, `Phase 1`, `Step 1` and `Pattern 1` in the heading text.
- 10 categories: `ai/`, `business-ideas/`, `containerization/`, `languages/`, `legacy/`, `misc/`, `roadmap/`, `security/`, `soft-skills/`, `system_design/`.
- Images live in `content/english/blog/{category}/img/{extension}/` and are referenced from blog posts as `../img/{extension}/name.ext`. The `../` is required because a post URL includes its slug directory. Mermaid sources live in `img/mmd/` and are published as non-page files.
- `summaryLength = 10` — excerpts truncate at 10 words.
- No Oxford comma (see `.github/instructions/blog.instructions.md`).
- Goldmark renders raw HTML in markdown (`unsafe = true`).
- Code highlighting: `guessSyntax = true`, `solarized-light` style.
- **Blog post dates must always be yesterday's date** (not today's), using the `+10:00` offset. Dates in a batch must be unique and at least 5 seconds apart — repeated timestamps collide and produce the GitHub Pages 404s documented in `README.md`.
- **Do not create `_index.md` in sub-section directories** (e.g. `blog/ai/_index.md`) — it makes Hugo treat the sub-section as separate from the parent `blog/`, excluding those posts from the main `/blog/` listing. Only `content/english/blog/_index.md` should exist.

## Architecture

- **Theme**: `github.com/zeon-studio/hugoplate` is **vendored** in `themes/hugoplate/` and tracked (115 files). `hugo.toml` sets `theme = "hugoplate"`, so the local copy is the active source, not the module import. Edit theme templates and styles in `themes/hugoplate/`, not via `config/_default/module.toml`, which only declares the theme plus helper modules (search, mermaid, icons, and so on).
- **CSS pipeline**: TailwindCSS v4 runs through Hugo's `css.TailwindCSS` pipe in `themes/hugoplate/layouts/_partials/essentials/style.html`. The entry stylesheet is `themes/hugoplate/assets/css/main.css`, which imports `custom.css`; the project file `assets/css/custom.css` shadows the theme placeholder, and the theme placeholder itself says never to write styles there. There is no `tailwind.config.js` and no PostCSS. Tailwind plugins (`tailwind-plugin/tw-theme.js`, `tw-bs-grid.js`) are loaded with `@plugin` directives from `main.css`.
- **Custom layouts**: only `layouts/shortcodes/` (`iframe.html`, `include.html`); everything else is inherited from the theme. `iframe.html` relies on the `.responsive-iframe` rules in `assets/css/custom.css`.
- **Mermaid**: `[markup.mermaid] enable = true` in `hugo.toml`; rendered pages load `mermaid.esm.min.mjs` from jsDelivr.
- **Required**: Hugo extended >= 0.151.0 (`config/_default/module.toml`), plus Go and Node.js locally (CI uses Node 24, no `.nvmrc`).
- **Generated, not committed**: `public/` and `resources/_gen/`. Do not edit files there.
- **Quirk**: `hugo_stats.json` is mounted with `disableWatch = true`, so CSS rebuilds do not trigger on stat changes alone.

## See also

- `.github/copilot-instructions.md` — writing style, anti-hallucination rules, fuller command table
- `.github/instructions/blog.instructions.md` — blog editorial rules
- `.github/commit-instructions.md` — commit message format
- `.github/skills/mermaid-to-png/SKILL.md` — converting `.mmd` files to PNG with `mmdc`
- `.github/agents/` — Codex agent definitions (not OpenCode)
