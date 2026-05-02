# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a static site generator for GitHub Scout — publishes AI-curated GitHub repo recommendations via RSS and a Zola-built blog. Forked from [yihong0618/gitblog](https://github.com/yihong0618/gitblog) (originally 橘鸦AI早报).

**Deployed at**: `https://2538514844.github.io/`  
**RSS**: `https://2538514844.github.io/rss.xml`

## Architecture

```
BACKUP/*.md          →  gen_zola.py  →  output/content/*.md  →  zola build  →  GitHub Pages
rss.xml (feedgen)    →  cp to output/public/
static/custom.css    →  injected into all HTML pages
```

The site pipeline runs exclusively in **GitHub Actions** (`.github/workflows/generate_site.yml`). There is no local dev server.

### Key files

| File | Role |
|------|------|
| `main.py` | Reads GitHub Issues → generates README.md, RSS (feedgen), and BACKUP/*.md backup files. Only issues owned by the authenticated user are processed. |
| `gen_zola.py` | Reads `BACKUP/*.md` → converts to Zola-compatible content with TOML frontmatter (including even-theme-required `reactions` field). Output goes to `output/content/`. |
| `config.toml` | Zola config: `even` theme, site title "GitHub Scout 每日精选", base URL, menu. |
| `static/custom.css` | ~1200 lines of custom CSS injected into every generated HTML page. Orange accent (`#c96442`), cream background, serif fonts, dark mode support. |
| `static/custom.js` | TOC fab button for mobile navigation. |
| `.github/workflows/generate_site.yml` | CI pipeline: run `main.py --skip-issues-rss`, run `gen_zola.py`, download Zola + even theme, build, inject custom.css, deploy to Pages. |

### Content flow

1. **github-scout** (separate Electron project) writes `rss.xml` + `BACKUP/*.md` to this directory
2. Auto `git push` triggers the deploy workflow
3. Workflow runs `main.py --skip-issues-rss` (skips Issues-based RSS, but still generates README)
4. `gen_zola.py` converts markdown to Zola format
5. Zola builds static site with even theme + custom.css injection
6. `rss.xml` is copied into the output (overwriting Zola's generated feed)

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
