# Copilot Tokens & Model Selection — Marp Slides

A Marp slide deck (in `index.md`) on GitHub Copilot tokens, token sizing, and model selection. Pushing to `main` automatically builds the deck and publishes it to **GitHub Pages** as live HTML slides, with a PDF and PPTX exported alongside.

## What's in here

```
.
├── index.md                       # the slide deck (Marp markdown)
├── .marprc.yml                    # optional repo-wide Marp config
├── .github/workflows/slides.yml   # CI: build + deploy to GitHub Pages
└── README.md
```

## One-time setup

1. **Create a repo and push these files.** Keep `index.md` at the repo root so the published site opens directly on the deck.
2. **Enable Pages with Actions as the source.** In the repo: **Settings → Pages → Build and deployment → Source: GitHub Actions**.
3. **Push to `main`** (or run the workflow manually from the **Actions** tab → *Build and Deploy Marp Slides* → *Run workflow*).

When the workflow finishes, the run summary shows the **page URL**. That URL serves the slides — navigate with the arrow keys / spacebar.

## Outputs

After a successful run, the published site contains:

- `/` (`index.html`) — the interactive HTML slide deck
- `/slides.pdf` — a PDF export
- `/slides.pptx` — an editable PowerPoint export

(The PDF/PPTX steps are best-effort: if the runner's browser export ever fails, the HTML deck still deploys.)

## Editing the deck

- Each slide is separated by `---` in `index.md`.
- Front matter at the top (`marp: true`, `theme: default`, `paginate: true`) controls rendering.
- For a live side-by-side preview while editing, install the **Marp for VS Code** extension and open `index.md`.

## Build locally (optional)

With Node.js 18+ installed:

```bash
# live preview server with hot reload
npx @marp-team/marp-cli@latest -s .

# one-off exports
npx @marp-team/marp-cli@latest index.md --html --output dist/index.html
npx @marp-team/marp-cli@latest index.md --pdf  --allow-local-files
npx @marp-team/marp-cli@latest index.md --pptx --allow-local-files
```

## Switching themes

Set `theme:` in `index.md` (or `.marprc.yml`) to `default`, `gaia`, or `uncover`. To use a custom theme, add a CSS file (e.g. `themes/mytheme.css` with a `/* @theme mytheme */` header), reference it in `.marprc.yml` via `themeSet`, and set `theme: mytheme`.
