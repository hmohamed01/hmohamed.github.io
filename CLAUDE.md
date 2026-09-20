# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

A Jekyll site on GitHub Pages — Hatem's notes on Windows endpoint management (Intune, ConfigMgr, PowerShell). Live at https://hmohamed01.github.io.

## There is no local build

There is no `Gemfile`, no test suite, and no linter. `actions/jekyll-build-pages` supplies the `github-pages` gem bundle in CI, so nothing is installed locally and `bundle exec jekyll serve` will not work without first creating a `Gemfile`. Don't offer to "run the tests" or "start the dev server" — neither exists.

**Pushing to `main` is the build, the deploy, and the only way to see a change rendered.**

```bash
git push origin main
```

### Watching the right deploy

`gh run list --limit 1` immediately after a push returns the *previous* run — GitHub hasn't registered the new one yet, and it will report a green status for the wrong commit. Always match on the commit SHA:

```bash
SHA=$(git rev-parse --short HEAD)
for i in 1 2 3 4 5 6 7 8; do
  RUN=$(gh run list --limit 3 --json databaseId,headSha \
        --jq ".[] | select(.headSha[0:7]==\"$SHA\") | .databaseId")
  [ -n "$RUN" ] && break
done
gh run watch "$RUN" --exit-status
```

### Verify the served output, not just the build

A green deploy does not mean the page is right. Broken front matter can ship YAML as visible body text, and a stale artifact can serve while the build succeeds. Check what's actually on the wire:

```bash
curl -s https://hmohamed01.github.io/ | grep -c 'layout: default'   # raw YAML leaked into body
curl -s https://hmohamed01.github.io/favicon.svg | diff -q - favicon.svg
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' https://hmohamed01.github.io/favicon-32.png
```

Content type matters as much as the status code — a favicon served as `text/plain` returns 200 and is silently ignored by browsers.

## Theme customization: one hook, deliberately

The theme is remote (`remote_theme: pages-themes/modernist@v0.2.0`); its files are not in this repo. Its layout ends `<head>` with `{% include head-custom.html %}`, and a local `_includes/head-custom.html` overrides the theme's empty one. **All custom icons, fonts, CSS, and JS live in that single file.** Add to it rather than introducing new asset files.

`_layouts/default.html` was forked once to inject markup into the header banner, then reverted. Forking means losing upstream fixes and owning ~50 lines of someone else's markup. Before reaching for it, check whether CSS can do the job from the content `<section>`:

- `section` is `padding: 15px 20px`, so `margin: -15px -20px` on a block breaks it full-bleed to the wrapper edges. This is how the homepage links band sits flush under the header without touching the layout.
- The theme's `pre` is already `margin: 0 -20px`, so code blocks bleed horizontally on their own.

If a layout override ever becomes genuinely unavoidable, copy it from the pinned tag (`v0.2.0`), never `master`.

To override the theme stylesheet instead, `assets/css/style.scss` needs empty front matter delimiters (`---` twice) or Jekyll treats it as static and never compiles the SCSS.

`layout: default` is the theme's only layout. `home`, `page`, and `post` produce an unstyled page and a build *warning* — the deploy still goes green, so the failure is easy to miss.

## Visual identity

A terminal/console direction drawn from the site's PowerShell subject matter. Derive new UI from these tokens rather than inventing a palette:

| Token | Hex | Use |
|---|---|---|
| console navy | `#0A3D7A` → `#011E45` | band and tile gradients |
| prompt cyan | `#7BE8FF` | chevron, `$variable`, hover, focus ring |
| link ice | `#C9E9FF` | link text at rest |
| key white | `#E3F0FF` | hashtable keys |
| cursor azure | `#3D8FE0` | block cursor, punctuation |

Type is **Cascadia Code** (Microsoft's terminal typeface), loaded from Google Fonts and used only in console-styled elements — body type stays the theme's. The favicon is the same chevron-and-cursor mark; `favicon.svg` is the source of truth and the PNGs are rendered from it with `rsvg-convert` (the apple-touch-icon comes from a separate full-bleed variant with no rounded corners, since iOS applies its own mask).

Rejected directions, so they don't get re-proposed: amber-on-slate, and Microsoft Fluent blue `#0078D4` (borrows Microsoft's identity rather than building Hatem's).

## Page types

**Markdown pages** (`index.md`, `csp-anatomy.md`) — front matter with `layout: default` and `title`, at the repo root.

**Standalone HTML** (`intune-csp-flow.html`, `oma-uri-tree.html`) — self-contained interactive documents with their own styling, carrying **no front matter**. Jekyll copies them verbatim as static files. Adding front matter would run them through Liquid, which breaks any file containing `{{` or `{%` — check before converting one.

Verify a static page survived intact by diffing the served copy against the source.

## Importing notes from the Obsidian vault

Source notes live in `~/Documents/MyVault/CSP-MDM-Deep-Dive/`. Obsidian syntax does not survive kramdown and must be converted on import:

| Obsidian | Convert to | Why |
|---|---|---|
| `[[Target]]`, `[[Target\|alias]]` | plain text, alias preserved | the targets aren't pages on this site and would 404 |
| `> [!info] Title` | `> **Title**` | kramdown renders the marker literally as `[!info]` |
| `tags:` front matter | keep, add `layout` + `title` | harmless, preserves the note's metadata |

The escaped-pipe form `[[X\|alias]]` appears inside table cells — unwrapping the wikilink must remove the `\|` along with it, or the table's column count breaks.

Re-link cross-references only if the target note has also been imported.

## Notes

- Verification screenshots are possible without a browser: `qlmanage -t -s 900 -o . page.html` renders HTML through WebKit. It blocks remote stylesheets and fonts, so inline the theme CSS into a scratch copy first; webfonts will still fall back to system mono.
- The README's "Current state" section drifts easily — it currently claims `index.md` carries theme demo boilerplate, which is no longer true. Update it when page content changes.
