# Well Designed System Website

This repository builds the Well Designed System technical blog and website with Hugo and deploys it to GitHub Pages. Site content is under `content/english/`; configuration lives in `hugo.toml` and `config/`.

## Build, preview, and formatting

| Command                                                | Purpose                                                                                            |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `npm run dev`                                          | Run Hugo's local development server. Drafts are excluded by default.                               |
| `hugo server -D`                                       | Run the development server with draft content.                                                     |
| `npm run build`                                        | Build the production site with garbage collection, minification, and Hugo template metrics.        |
| `npm run preview`                                      | Run a production-mode local preview.                                                               |
| `npm run format`                                       | Format the repository with Prettier (writes changes).                                              |
| `npx prettier --check .github/copilot-instructions.md` | Check formatting for one file without writing changes. Replace the path to check a different file. |
| `npm run update-modules`                               | Update Hugo module dependencies.                                                                   |

`package.json` defines no test suite, test command, or linter, so there is no single-test command. Run `npm run build` to validate site generation. CI installs Node.js 24 dependencies with `npm ci` and builds with `hugo --gc --minify` plus the GitHub Pages base URL; see `.github/workflows/deploy.yml`. Hugo Extended 0.151.0 or later is required by `config/_default/module.toml`.

## Architecture

- Hugo uses `content/english/` as its English content directory. Blog posts are grouped under `content/english/blog/` by category, with shared site settings in `hugo.toml` and `config/_default/`.
- The site uses the Hugoplate theme and Hugo modules declared in `config/_default/module.toml`. Most templates come from the theme; project-specific layout overrides are in `layouts/shortcodes/`.
- The theme's CSS pipeline uses Tailwind CSS v4. Keep project-specific styles in `assets/css/custom.css` and project Tailwind plugins in `tailwind-plugin/`. The theme CSS scans `hugo_stats.json` for classes; that file is tracked and may change when content or templates introduce classes.
- `npm run build` and CI generate `public/`. Treat it as build output and do not edit it directly.

## Content conventions

- Blog posts use TOML front matter delimited by `+++`. Preserve the existing front matter format in other content files; section index files may use YAML.
- Create posts under `content/english/blog/<category>/<slug>.md`. The current categories are `ai/`, `business-ideas/`, `containerization/`, `languages/`, `legacy/`, `misc/`, `roadmap/`, `security/`, `soft-skills/` and `system_design/`.
- Keep post summaries to 10 words or fewer; Hugo's `summaryLength` is 10.
- Blog headings start at `##`. CSS numbers H2 and H3 headings automatically, so do not add section numbers solely to show hierarchy. Preserve meaningful labels such as “Part 1” or “Phase 1”. The table of contents is configured for ordered headings from H2 through H5.
- Follow the repository's blog date convention: use yesterday's date with the `+10:00` offset. For multiple posts in one change, use unique timestamps at least five seconds apart.
- Do not add `_index.md` files to blog category subdirectories; keep the blog section index at `content/english/blog/_index.md`.
- Follow nearby posts for filename, image placement, and shortcode patterns. Images commonly live in an `img/` directory beside posts in the same category.
- Do not use Oxford commas in blog content. When adding technical content, verify claims against official sources and include links. Keep statements precise and unambiguous.

## Related repository guidance

- [AGENTS.md](../AGENTS.md) contains additional repository conventions and deployment details.
- [Blog Markdown Instructions](./instructions/blog.instructions.md) covers blog-specific editorial rules.
- [Commit Message Instructions](./commit-instructions.md) defines commit message format.
- [README.md](../README.md) documents local-network development and the GitHub Pages 404 troubleshooting note.
