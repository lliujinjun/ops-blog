+++
date = '2026-07-04T13:14:00+08:00'
draft = false
title = '🚀 Deploy Hugo to GitHub Pages with GitHub Actions'
+++

## 📋 Overview

This blog is built with Hugo and hosted on GitHub Pages. Every time I push to `main`, a GitHub Action builds the site and deploys it automatically. This post documents how it's set up — including the submodule pitfalls I hit along the way.

---

## Architecture

```
git push to main
       │
       ▼
GitHub Actions workflow
       │
       ├── Checkout repo + submodules (themes)
       ├── Setup Hugo (extended)
       ├── hugo --minify → ./public/
       └── Deploy to GitHub Pages
              │
              ▼
       https://lliujinjun.github.io/ops-blog/
```

No manual `scp`, no branch switching, no separate server. Everything happens in CI. 🤖

---

## Step 1: Set `baseURL`

Your `hugo.toml` needs the correct URL so Hugo generates proper paths in RSS feeds and sitemaps:

```toml
# For a project page:
baseURL = 'https://<username>.github.io/<repo>/'

# For a user/org page:
baseURL = 'https://<username>.github.io/'

# For a custom domain:
baseURL = 'https://yourdomain.com/'
```

My repo is `lliujinjun/ops-blog`, so:

```toml
baseURL = 'https://lliujinjun.github.io/ops-blog/'
```

---

## Step 2: Create the GitHub Actions workflow

`.github/workflows/deploy.yml`:

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true    # ← THIS is critical
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true      # PaperMod needs Sass support

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Key points:
- `submodules: true` — fetches the theme repos (PaperMod in my case)
- `extended: true` — needed for PaperMod's Sass/SCSS processing
- Two jobs: `build` produces the artifact, `deploy` publishes it
- No `gh-pages` branch needed — GitHub Actions handles everything

---

## Step 3: Enable GitHub Pages

Go to your repo → **Settings** → **Pages**:
- Source: **GitHub Actions** ✨

No branch selection, no custom domain config needed unless you have one.

---

## 🔥 The Submodule Trap

First deploy failed with:

```
Error: fatal: No url found for submodule path 'themes/papermod' in .gitmodules
```

**What happened:** I had three submodules in my repo — `ananke`, `papermod`, and `coder`. But only `ananke` was listed in `.gitmodules`. The other two were only in the local `.git/config`:

```gitconfig
# .gitmodules (committed to repo)
[submodule "themes/ananke"]
    path = themes/ananke
    url = https://github.com/theNewDynamic/gohugo-theme-ananke.git

# .git/config (local only — NOT shared!)
[submodule "themes/papermod"]
    url = https://github.com/adityatelange/hugo-PaperMod
    active = true
[submodule "themes/coder"]
    url = https://github.com/luizdepra/hugo-coder
    active = true
```

**Fix:** Add the missing entries to `.gitmodules`:

```gitconfig
[submodule "themes/papermod"]
    path = themes/papermod
    url = https://github.com/adityatelange/hugo-PaperMod

[submodule "themes/coder"]
    path = themes/coder
    url = https://github.com/luizdepra/hugo-coder
```

After committing and pushing, the next workflow run fetched all submodules and built successfully. ✅

---

## ⚡ Auto-Deploy on Every Push

Since the workflow triggers on `push` to `main`, I just need to:

```bash
$ git add content/posts/my-new-post.md
$ git commit -m "add my new post"
$ git push origin main
```

Wait ~60 seconds, and the new post is live. The workflow handles:
- ✅ Checking out submodules
- ✅ Building with `hugo --minify`
- ✅ Uploading the artifact
- ✅ Deploying to GitHub Pages

No manual steps, no server management, no FTP nonsense. 🚀

---

## 📊 Summary

| Step | What I Did | Time |
|---|---|---|
| Set `baseURL` | `hugo.toml` → `https://<user>.github.io/<repo>/` | 10 seconds |
| Create workflow | `.github/workflows/deploy.yml` | 5 minutes |
| Enable Pages | Repo Settings → Pages → GitHub Actions | 30 seconds |
| Fix submodules | Add missing entries to `.gitmodules` | 2 minutes |
| First deploy | Push → Action runs → Site live | ~1 minute |

## 💡 Lessons Learned

1. **📜 `.gitmodules` is the contract** — submodules added with `git submodule add` should update `.gitmodules` automatically, but if you ever add them manually or clone in a weird way, double-check the file is committed.
2. **🧩 `submodules: true` is mandatory** — without it, your theme is gone on the build runner. `fetch-depth: 0` also helps with some Hugo features.
3. **🔧 `extended: true` for PaperMod** — Hugo Extended includes the Sass/SCSS pipeline. PaperMod and many other themes depend on it.
4. **⏱️ First deploy takes the longest** — subsequent pushes are faster because Docker layers and Hugo are cached.

---

*The blog is now CI/CD-enabled. Every commit to `main` = instant publication. 📝➡️🌐*
