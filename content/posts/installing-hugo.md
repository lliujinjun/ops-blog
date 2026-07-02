+++
date = '2026-07-01T23:09:42+08:00'
draft = false
title = '📝 Installing Hugo on Linux'
+++

## 📋 Overview

This post documents the manual installation of Hugo (latest version) on a Linux system, plus the creation of this very blog. 🔁

## 🤔 What is Hugo?

[Hugo](https://gohugo.io/) is a fast and flexible static site generator written in Go. It's popular among developers, ops engineers, and writers who want a simple, version-controlled way to publish content. ⚡

## 🛠️ Installation Steps

### 1. 🔍 Check the Latest Version

First, find the latest release from GitHub:

```bash
curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest | grep -oP '"tag_name": "\Kv[0-9.]+'
```

At the time of writing, the latest version is **v0.163.3**.

### 2. 📥 Download the Binary

Download the extended version (includes Sass/SCSS support):

```bash
curl -LO https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_linux-amd64.tar.gz
```

> **🌏 Note:** If direct GitHub downloads are slow or blocked in your region, you can use a mirror proxy like `ghfast.top`:
> ```bash
> curl -LO https://ghfast.top/https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_linux-amd64.tar.gz
> ```

### 3. 📂 Extract and Install

Extract the archive and move the binary to your local bin directory:

```bash
tar -xzf hugo_extended_0.163.3_linux-amd64.tar.gz
mkdir -p ~/.local/bin
mv hugo ~/.local/bin/
rm -f hugo_extended_0.163.3_linux-amd64.tar.gz README.md LICENSE
```

### 4. ✅ Verify Installation

```bash
~/.local/bin/hugo version
```

Expected output:
```
hugo v0.163.3-4d22555aebf458d5d150500c9ac4bee5b24cf0d3+extended linux/amd64 BuildDate=2026-06-18T16:18:24Z VendorInfo=gohugoio
```

### 5. 🧭 Add to PATH (Optional)

Add this to your `~/.bashrc` or `~/.zshrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 🆕 Creating a New Hugo Site

### 1. 🚀 Create the Site

```bash
hugo new site ops-blog
cd ops-blog
```

### 2. 📚 Initialize Git

```bash
git init
```

### 3. 🎨 Add a Theme

This blog uses the [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) theme:

```bash
git submodule add --depth 1 https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

### 4. ⚙️ Configure the Site

Edit `hugo.toml`:

```toml
baseURL = 'https://example.org/'
locale = 'en-us'
title = 'Ops Engineer Blog'
theme = 'ananke'
```

### 5. ✍️ Create Your First Post

```bash
hugo new content content/posts/installing-hugo.md
```

### 6. 👀 Start the Development Server

```bash
hugo server --buildDrafts
```

Visit `http://localhost:1313` to see your site. 🌐

## 🔄 Git Workflow

Since the Hugo project is Git-managed, here's a basic workflow:

```bash
# Stage changes
git add .

# Commit
git commit -m "Initial Hugo site setup"

# Push to remote (after setting up a remote repository)
git push origin main
```

## 💡 Why Hugo for Ops Engineers?

- ⚡ **Fast builds**: Hugo generates sites in milliseconds
- ✏️ **Markdown-native**: Write in Markdown, focus on content
- 📚 **Version controlled**: Your entire site is in Git
- 🔄 **CI/CD friendly**: Easy to deploy with GitHub Actions, GitLab CI, etc.
- 🗄️ **No database**: Just static files — easy to host anywhere

## 🔜 Next Steps

- Customize the theme 🎨
- Set up a CI/CD pipeline for automatic deployment 🚀
- Configure a custom domain 🌐
- Add analytics or comments 📊

---

*📝 This post was written as part of setting up this blog. Meta, right?* 🔁
