# hmohamed01.github.io

[![Deploy Jekyll](https://github.com/hmohamed01/hmohamed01.github.io/actions/workflows/jekyll-gh-pages.yml/badge.svg)](https://github.com/hmohamed01/hmohamed01.github.io/actions/workflows/jekyll-gh-pages.yml)
[![Pages](https://img.shields.io/website?url=https%3A%2F%2Fhmohamed01.github.io&label=site)](https://hmohamed01.github.io)

Personal site — notes on Windows endpoint management: Intune, ConfigMgr, and PowerShell.

**Live at https://hmohamed01.github.io**

## How it works

Jekyll, built and deployed by GitHub Actions on every push to `main`.

| | |
|---|---|
| Theme | [`pages-themes/modernist`](https://github.com/pages-themes/modernist), pinned to `v0.2.0` via `remote_theme` |
| Build | `actions/jekyll-build-pages` — ships the `github-pages` gem bundle, so no `Gemfile` is needed |
| Deploy | `actions/deploy-pages`, triggered on push to `main` |
| Site type | **User site.** The repo name matches `<owner>.github.io`, so it serves from the domain root and `baseurl` is empty |

## Files

```
_config.yml         site settings - title, description, url, theme
index.md            home page - must stay at the root
CSP-MDM-Deep-Dive/  topic folder; content lives in folders like this
favicon.svg         site icon (PNG fallbacks alongside it)
_includes/          head-custom.html - icons, fonts, custom CSS
.github/            the build and deploy workflow
```

## Adding a page

Content is organised in topic folders. Create a `.md` file inside one with front matter:

```markdown
---
layout: default
title: Page title
---

Content here.
```

**`layout: default` is the only layout this theme provides.** Modernist ships a single `_layouts/default.html` — `home`, `page` and `post` come from other themes and will render the page unstyled with only a build *warning*, not an error. The deploy will go green regardless, so the failure is easy to miss.

Link to it from the homepage with a root-absolute path, e.g. `[About](/Topic-Folder/about.html)`. Relative paths work too, but absolute ones stay correct no matter which page links to them.

## Editing the look

The theme's own stylesheet is inherited via `remote_theme`. To override it, create `assets/css/style.scss`:

```scss
---
---

@import "jekyll-theme-modernist";

/* overrides below */
```

The empty front matter delimiters are required — without them Jekyll treats the file as static and does not compile the SCSS.

## Current state

`index.md` is the links block plus a list of reference pages. `CSP-MDM-Deep-Dive/` holds the first topic: `csp-anatomy.md` imported from the Obsidian vault, plus `intune-csp-flow.html` and `oma-uri-tree.html`, which are self-contained interactive pages.
