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
_config.yml      site settings - title, description, url, theme
index.md         home page
another-page.md  demo second page, linked from index
.github/         the build and deploy workflow
```

## Adding a page

Create a `.md` file at the repo root with front matter:

```markdown
---
layout: default
title: Page title
---

Content here.
```

**`layout: default` is the only layout this theme provides.** Modernist ships a single `_layouts/default.html` — `home`, `page` and `post` come from other themes and will render the page unstyled with only a build *warning*, not an error. The deploy will go green regardless, so the failure is easy to miss.

Link between pages with a relative path, e.g. `[About](./about.html)`.

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

The content of `index.md` and `another-page.md` is the theme's demo boilerplate, kept deliberately so every styled element is visible. Replace it with real content and delete `another-page.md` when it stops being useful.
