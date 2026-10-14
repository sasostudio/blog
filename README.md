# Saso Studio — portfolio hub & dev log

Source for [sasostudio.dev](https://sasostudio.dev) — studio intro, selected work (case studies), and a build-in-public dev log, all in one site.
`saso-studio.vercel.app` permanently redirects here (`vercel.json`), except `/app-ads.txt` and the Search Console verification file, which stay served at the old host.

Custom Jekyll site (no theme gem). Deployed by **Vercel** (project `saso-studio`, repo-connected, push to `main` → production).
The older address chpark03.github.io no longer resolves — the GitHub account was renamed to `sasostudio` (2026-10-14), and GitHub does not redirect Pages sites across a rename, so the redirect stubs and the Pages workflow were retired.

## Structure

- `index.md` — home (studio intro + selected work + recent dev log)
- `_posts/YYYY-MM-DD-slug.md` — dev log posts (Korean-first, English version included)
- `_projects/*.md` — portfolio case studies (front-matter driven: `steps`, `status`, `links`)
- `archive.md` — `/archive/` page listing closed projects (`archived: true` in `_projects/`)
- `_layouts/` — custom layouts (`default`, `home`, `post`, `page`, `project`, `loglist`, `archivelist`)
- `_includes/` — `log-item`, `work-item` (project row), `play-buttons` (store buttons from a project's `play:` list)
- Post front matter: `project:` (links the post to a `_projects/` card) · optional `description:` (summary box + SEO/share description). Posts keep the KO → `---` → `## English version` structure; the post layout splits on it for the KO/EN switch.
- `assets/css/main.css` — full design system ("검증 원장" ledger: paper light single theme; seal-stamp verdict motif)
- `vercel.json` — Vercel build config (`bundle exec jekyll build` → `_site`) + old-host redirect
- `feed.xml` — copy of the jekyll-feed template with entry IDs pinned to `feed.id_base` (`_config.yml`), so a domain change never re-surfaces old posts in feed readers

## Local preview

```powershell
bundle install
bundle exec jekyll serve
```
