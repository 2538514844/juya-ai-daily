# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a static site generator for GitHub Scout — publishes AI-curated GitHub repo recommendations via RSS and a Zola-built blog. Forked from [yihong0618/gitblog](https://github.com/yihong0618/gitblog) (originally 橘鸦AI早报).

**Deployed at**: `https://2538514844.github.io/`  
**RSS**: `https://2538514844.github.io/rss.xml`

## Setup

```bash
pip install -r requirements.txt
```

Python >= 3.13 required. `pyproject.toml` declares `pdm-backend` as the build system, but CI and local usage both go through `pip install -r requirements.txt`.

## Local preview

```bash
bash scripts/preview_site_local.sh
```

Serves the full site at `http://127.0.0.1:4173/juya-ai-daily/`. Downloads `isite` + `zola` binaries into `.local-tools/`, runs the site generation pipeline, and starts a Python HTTP server. Key options:

| Flag | Purpose |
|------|---------|
| `--skip-generate` | Reuse existing `output/public/` (no GitHub API calls) |
| `--snapshot-live` | Download the live site via `wget --mirror` instead of rebuilding |
| `--no-serve` | Build only, don't start the HTTP server |
| `--port <N>` | Change the preview port (default 4173) |
| `--force-redownload` | Redownload the isite/zola binaries |

The script also sets up CSS/JS live-reload: changes to `static/custom.css` are hot-replaced in the browser; changes to `static/custom.js` trigger a full page refresh.

## Architecture

```
BACKUP/*.md          →  gen_zola.py  →  output/content/*.md  →  zola build  →  GitHub Pages
rss.xml (feedgen)    →  cp to output/public/
static/custom.css    →  injected into all HTML pages
```

The primary site pipeline runs in **GitHub Actions** via two workflows:

- **`generate_site.yml`**: Full site deploy. Triggered by pushes to `master` (when relevant files change), issue/comment events, and manual dispatch. Runs `main.py --skip-issues-rss` (skips RSS-from-Issues because `rss.xml` comes from github-scout), then `gen_zola.py`, then Zola build + custom.css injection + deploy to GitHub Pages. Requires `contents: read`, `issues: read`, `pages: write`, `id-token: write`.
- **`generate_readme.yml`**: README-only update. Triggered by issues, comments, pushes to `main.py`, and manual dispatch. Runs `main.py` **without** `--skip-issues-rss` (generates RSS from Issues and saves Issues as BACKUP/*.md), then commits/pushes the results. Requires `contents: write` and the `G_T` secret for GitHub API access.

A local preview is available via `scripts/preview_site_local.sh`.

### Key files

| File | Role |
|------|------|
| `main.py` | Reads GitHub Issues → generates README.md, RSS (feedgen), and BACKUP/*.md backup files. Only issues owned by the authenticated user are processed. |
| `gen_zola.py` | Reads `BACKUP/*.md` → converts to Zola-compatible content with TOML frontmatter (including even-theme-required `reactions` field). Output goes to `output/content/`. |
| `config.toml` | Zola config: `even` theme, site title "GitHub Scout 每日精选", base URL, menu. |
| `static/custom.css` | ~1200 lines of custom CSS injected into every generated HTML page. Orange accent (`#c96442`), cream background, serif fonts, dark mode support. |
| `static/custom.js` | TOC fab button for mobile navigation. |
| `static/icon.png` | Site icon (144px), referenced in RSS feed and injected into page `<head>`. |
| `requirements.txt` | Python dependencies: PyGithub, feedgen, marko, markdown. CI installs from this file. |
| `.github/workflows/generate_site.yml` | Primary CI: `main.py --skip-issues-rss` → `gen_zola.py` → Zola build → custom.css/js injection → deploy to Pages. |
| `.github/workflows/generate_readme.yml` | Secondary CI: `main.py` (with RSS from Issues) → commit/push README + BACKUP/*.md. |
| `scripts/preview_site_local.sh` | Local dev server: downloads isite + zola, runs the full pipeline, serves with CSS/JS live-reload. |

### Content flow

1. **github-scout** (separate Electron project) writes `rss.xml` + `BACKUP/*.md` to this directory
2. Auto `git push` triggers the `generate_site.yml` deploy workflow
3. Workflow runs `main.py --skip-issues-rss` (skips Issues-based RSS, but still regenerates README)
4. `gen_zola.py` converts markdown to Zola format
5. Zola builds static site with even theme + custom.css/js/icon injection
6. `rss.xml` is copied into the output (overwriting Zola's generated feed)

Separately, `generate_readme.yml` runs `main.py` without `--skip-issues-rss` when Issues change — this generates RSS from Issues, saves Issues to BACKUP/*.md, and commits the results back to the repo.

### `main.py` parameters

```
python main.py <github_token> <repo_name> [--issue_number N] [--skip-issues-rss]
```

- `--skip-issues-rss`: Skips RSS generation from Issues (RSS comes from github-scout instead)
- Without it, `main.py` generates RSS from repo Issues and saves Issues as BACKUP/*.md

### `gen_zola.py`

- Expects files named `{number}_{YYYY-MM-DD}.md` in BACKUP/
- Extracts title from `# [title](url)` pattern
- Extracts tags from `## 标签` section with backtick-wrapped items
- Extracts date from filename
- Generates Zola frontmatter with even-theme-compatible `[extra].reactions`

### BACKUP/ directory

Content is generated externally (by github-scout). The deploy workflow triggers when BACKUP/ files change. Each `.md` file becomes one Zola page. The `.gitkeep` file must remain to keep the directory in git.
