# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal technical blog ("Search and relevance decoded") built on the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll theme via the Chirpy Starter template. The theme itself ships as the `jekyll-theme-chirpy` gem — this repo only holds the site's content and the minimal files Chirpy needs present (`_config.yml`, `_plugins`, `_tabs`, `index.html`). Layouts, includes, sass, and most assets live inside the gem, not here.

## Commands

- `bundle install` — install dependencies (run once, and after any Gemfile change).
- `bash tools/run.sh` — serve locally with livereload at `http://127.0.0.1:4000`. Add `-p` for production mode, `-H <host>` to change the bind host.
- `bash tools/test.sh` — production build into `_site/` then run html-proofer link/HTML validation. This is the CI check; run it before pushing. Accepts `-c "_config.yml,_config2.yml"` for layered configs.
- `bundle exec jekyll b` — plain build without the html-proofer pass.

There is no separate lint step; `tools/test.sh` (html-proofer) is the validation gate.

## Content authoring

- Posts go in `_posts/` named `YYYY-MM-DD-title.md` with Chirpy front matter (`title`, `date`, `categories`, `tags`). `_posts/` is currently empty — the site is freshly scaffolded.
- Posts resolve to the permalink `/posts/:title/` (set in `_config.yml`), not the dated path.
- `_tabs/` holds the sidebar nav pages (about, archives, categories, tags); each has an `order` and `icon` in its front matter. Tabs render with `layout: page` and permalink `/:title/`.
- `_drafts/` posts have comments disabled by default.

## Architecture notes worth knowing

- **`_plugins/posts-lastmod-hook.rb`** sets each post's `last_modified_at` from git history (`git log` on the post's file). Because it shells out to git, the "last modified" timestamps depend on real commit history — a shallow clone or squashed history changes them. CI checks out with `fetch-depth: 0` for this reason.
- **`assets/lib`** is a git submodule (`chirpy-static-assets`, self-hosted JS/CSS libs). It is currently *not* fetched in CI (the `submodules: true` line in the workflow is commented out); the site falls back to CDN-hosted libs. If you enable `assets.self_host` in `_config.yml`, you must also uncomment submodule checkout in `.github/workflows/pages-deploy.yml`.
- **Deployment** is fully automated: pushing to `main` triggers `.github/workflows/pages-deploy.yml`, which builds with `JEKYLL_ENV=production`, runs html-proofer, and deploys to GitHub Pages. There is no manual deploy step.
- **`jekyll-archives`** generates the `/categories/:name/` and `/tags/:name/` pages at build time — these are not real files in the repo.
- Site-wide identity, SEO, analytics, comment providers (disqus/utterances/giscus), and PWA settings are all driven from `_config.yml`. Changing `_config.yml` requires restarting `tools/run.sh` (Jekyll does not hot-reload config).
