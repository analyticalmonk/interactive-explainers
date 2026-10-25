# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Interactive Explainers is a collection of interactive, educational articles inspired by distill.pub. It is a **pure HTML/CSS/vanilla JS project** with zero dependencies, no build system, no bundler, and no package manager.

## Development

There is no build step. To develop locally, serve the files with any static HTTP server:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000`. There are no tests or linters. Verify in a browser: zero console errors, every figure control exercised, layout intact at mobile width.

## Architecture

- **Landing page** (`index.html`) - Newest-first list of `li.article-entry` rows. Each row has its own animated thumbnail (`canvas#thumb-<slug>`, 480x280) drawn by its own IIFE in the bottom `<script>`.
- **Explainer articles** (`<topic>/index.html`) - Fully self-contained HTML files. No CSS or JS is shared between articles. Two-column layout: sticky sidebar nav + main content with interactive Canvas figures.

### Interactive Figures Pattern

Each interactive visualization is a self-contained IIFE using HTML5 Canvas:

```javascript
(function() {
  const init = initCanvas('canvas-id');
  if (!init) return;
  const { canvas, ctx, w, h } = init;
  let state = { /* ... */ };
  function draw() { /* render logic */ }
  function animate() { requestAnimationFrame(animate); draw(); }
  canvas.addEventListener('mousedown', /* ... */);
  animate();
})();
```

Each article defines its own inline `initCanvas(id)` for DPR scaling. Copy the current version from `superpowers/index.html` (it returns `null` for a missing canvas). All rendering is hand-written Canvas 2D with no libraries.

### Styling Conventions

- Serif body text (`Crimson Pro`), monospace for code (`JetBrains Mono`), system sans-serif for UI
- Each article defines its own palette in `:root` (`--bg`, `--accent`, topic colors). There is no shared palette.
- Content max-width: 720px, sidebar: 220px
- Article breakpoints at 1060px (sidebar hides) and 600px (mobile layout). The landing page uses one at 700px.
- CSS custom properties for theming, BEM-adjacent class naming (`article-`, `figure-`, `ctrl-`)

## Adding a New Explainer

Model new articles on `superpowers/`, the best-received explainer, with `codex-embedding/` and `stagehand/` as secondary references. `dspy/` and `autoresearch/` are older outliers. Follow the repo's own conventions, not a generic explainer template.

1. Create `<slug>/index.html`. Copy the scaffold from a reference article: CSS, `initCanvas`, scroll-spy, site header with `../` back link, and the GoatCounter snippet in `<head>`.
2. Header: `div.overline` + `h1` + `p.lead` + `div.meta` ("Month YYYY &middot; N min read &middot; N figures"). Each figure opens with `div.figure-label` ("Figure N - Title"). Break long h2 sections into h3 subsections. End with `footer.article-footer` containing a **References:** line.
3. Add a row at the top of the list in root `index.html`, plus its `thumb-<slug>` thumbnail IIFE.
4. Add an entry at the top of `## Articles` in `README.md`: `- **[Title](slug/)** - One-sentence summary. N interactive figures.`

Keep the title identical in `<title>`, `<h1>`, the landing row, and the README. Keep the figure count in sync across the article meta, the landing row, and the README.

## Gotchas

- Article dirs often hold untracked debug screenshots/recordings (`*.png`, `*.mp4`) that nothing references. Stage files by name; don't `git add -A`.

## Git

- Branches: `feat/<slug>-explainer`, `fix/<topic>`, `style/<topic>`, merged to `main` via GitHub PRs.
- Commit messages: imperative, sentence case, no `feat:`-style prefix, no trailing period.
