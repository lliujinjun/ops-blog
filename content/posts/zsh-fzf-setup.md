+++
date = '2026-07-01T23:45:00+08:00'
draft = false
title = 'Oh My Zsh + Plugins + fzf Shell Integration'
+++

## Overview

This guide covers setting up a modern Zsh environment on a Linux server:
- **Oh My Zsh** — framework for managing Zsh configuration
- **zsh-syntax-highlighting** — colorizes commands as you type
- **zsh-autosuggestions** — suggests commands based on history
- **fzf** — fuzzy finder with shell integration

## Prerequisites

- A Linux server (tested on CentOS/RHEL/Fedora family)
- Sudo or root access for package installs
- Zsh installed: `which zsh` should return a path

If Zsh isn't installed:

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y zsh

# Ubuntu/Debian
sudo apt install -y zsh
```

## Step 1: Install Oh My Zsh

Oh My Zsh is installed via a curl script from the project's GitHub:

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

After running this:
1. You'll be prompted to change your default shell to Zsh — say **yes**
2. Your `~/.zshrc` will be populated with the Oh My Zsh default configuration

### What this sets up:

| Item | Location |
|---|---|
| Oh My Zsh framework | `~/.oh-my-zsh/` |
| Zsh config file | `~/.zshrc` |
| Custom plugins directory | `~/.oh-my-zsh/custom/plugins/` |
| Custom themes directory | `~/.oh-my-zsh/custom/themes/` |

## Step 2: Install zsh-syntax-highlighting

This plugin colorizes commands green/red as you type — instant feedback on whether a command exists.

Clone it into Oh My Zsh's custom plugins directory:

```bash
git clone --depth 1 https://github.com/zsh-users/zsh-syntax-highlighting.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

## Step 3: Install zsh-autosuggestions

This plugin suggests completions based on your command history, shown as faint gray text. Press **→** to accept a suggestion.

```bash
git clone --depth 1 https://github.com/zsh-users/zsh-autosuggestions.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

## Step 4: Enable Plugins in ~/.zshrc

Edit `~/.zshrc` and find the line that starts with `plugins=(git)`.

Replace it with:

```bash
plugins=(
  git
  zsh-syntax-highlighting
  zsh-autosuggestions
)
```

**Important ordering note:** `zsh-syntax-highlighting` **must** be the last plugin in the list, or syntax highlighting may not work correctly for suggestions.

## Step 5: Install fzf — Fuzzy Finder

### 5a. Install the binary

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y fzf

# Ubuntu/Debian
sudo apt install -y fzf

# Or install from source (latest version)
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

### 5b. fzf Installation Prompts

The `~/.fzf/install` script will ask three questions:

```
❯ Do you want to enable fuzzy auto-completion? (y/n)  →  y
❯ Do you want to enable key bindings? (y/n)            →  y
❯ Do you want to update your shell configuration? (y/n)  →  y
```

Answer **y** to all three for full shell integration.

### What fzf provides:

| Feature | Description |
|---|---|
| **Ctrl+R** | Fuzzy-search through command history |
| **Ctrl+T** | Fuzzy-search files/directories to paste into command line |
| **Alt+C** | Fuzzy-find a directory and `cd` into it |
| **Auto-completion** | Type `**` + Tab to trigger fuzzy completion (e.g., `ssh **<Tab>`) |

## Step 6: Reload Your Shell

```bash
exec zsh
```

Or source the config:

```bash
source ~/.zshrc
```

## Verification

After reloading, test each component:

```bash
# 1. Oh My Zsh — check theme prompt
echo $ZSH_VERSION

# 2. Syntax highlighting — type a valid command (should be green)
ls /tmp

# 3. Auto-suggestions — type a partial command you've run before (g should show git...)

# 4. fzf — press Ctrl+R, start typing to fuzzy-search history
```

## Final ~/.zshrc

Here's what your plugins section should look like:

```bash
# Path to your Oh My Zsh installation
export ZSH="$HOME/.oh-my-zsh"

# Theme
ZSH_THEME="robbyrussell"

# Plugins
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting    # MUST be last
)

source $ZSH/oh-my-zsh.sh

# fzf shell integration (added by ~/.fzf/install)
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh
```

## Troubleshooting

| Issue | Fix |
|---|---|
| Syntax highlighting not working | Move `zsh-syntax-highlighting` to the **end** of the plugins array |
| Auto-suggestions showing wrong commands | Clear history with `cat /dev/null > ~/.zsh_history` |
| fzf Ctrl+T/Ctrl+R not working | Ensure `~/.fzf.zsh` is sourced in `~/.zshrc` |
| Slow shell startup | Check plugins count; `git clone --depth 1` keeps repos small |

## Summary

```
Before (bash):     $ ls <--- plain boring text
After (zsh+ohmy):  ✨ $ ls  ← green, with suggestions in gray
                    Also: Ctrl+R = fuzzy history search 🚀
```

Everything is managed in `~/.zshrc` — version-control it and clone the same setup onto any machine.
