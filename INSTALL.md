# Hugo Installation Record

**Date:** 2026-07-01
**Version:** v0.163.3 (extended)
**System:** Linux x86_64 (WSL2)

## Method

Manual binary installation (not via package manager).

## Steps

1. Identified latest version: v0.163.3
2. Downloaded `hugo_extended_0.163.3_linux-amd64.tar.gz` from GitHub releases
3. Extracted and moved `hugo` to `~/.local/bin/`
4. Verified: `hugo v0.163.3+extended linux/amd64`

## Notes

- Direct GitHub downloads were slow due to network conditions; used a mirror proxy.
- Hugo extended version required for Sass/SCSS support.

## Site Created

- Project: `ops-blog/` at workspace root
- Theme: Ananke (git submodule)
- First post: `content/posts/installing-hugo.md`
