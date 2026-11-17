---
name: adding-blog-post
description: Use when adding a new blog post to the bpradipt/blogs Jekyll site, especially when the draft contains mermaid diagrams or markdown tables to render as PNG images, or when a post needs front matter, a dated filename, and cross-post-safe image references.
---

# Adding a blog post

## Overview

This Jekyll blog serves figures as committed PNGs so the site stays JavaScript-free and figures survive cross-posting (Medium/Dev.to don't render mermaid or markdown tables). Convert mermaid diagrams and markdown tables to PNG locally, commit the images, and reference them with `relative_url`.

## Workflow

1. **Find the draft** in `_posts/` — often an undated file (e.g. `my-post.md`) with no front matter.
2. **Front matter + filename.** Add YAML front matter (`title`, `tags`); remove any top-level `# H1` (the layout renders the title). Rename to `_posts/YYYY-MM-DD-slug.md`.
3. **Mermaid → PNG.** For each ` ```mermaid ` block:
   - Save the diagram source (between the fences) to `assets/diagrams/<name>.mmd`.
   - Render: `./scripts/render-mermaid.sh assets/diagrams/<name>.mmd assets/diagrams/<name>.png`
   - Replace the block with `![<caption>]({{ '/assets/diagrams/<name>.png' | relative_url }})`.
4. **Tables → PNG.** For each markdown table:
   - Save the header, `|---|` separator, and data rows to `assets/diagrams/tables/<name>.md`.
   - Render: `python3 scripts/render-table.py assets/diagrams/tables/<name>.md assets/diagrams/<name>.png`
   - Replace the table with the same `![...]` image form.
5. **Commit the sources too.** Keep the `.mmd` and `tables/*.md` files next to the PNGs so figures can be regenerated.
6. **Cross-links.** Link to other posts by URL (`{{ '/YYYY-MM-DD/slug/' | relative_url }}`), not raw `.md` filenames.
7. **External figures** (draw.io etc.): use a PNG in `assets/diagrams/`, referenced with `relative_url`.

## Quick reference

| Input | Command | Output |
|---|---|---|
| mermaid `.mmd` | `./scripts/render-mermaid.sh <in>.mmd <out>.png` | PNG |
| table `.md` | `python3 scripts/render-table.py <in>.md <out>.png` | PNG |

Prereqs: Node/npx (first mermaid run downloads mermaid-cli + headless Chromium — slow once); Pillow (`pip3 install pillow`) for tables.

## Gotchas

- **PNG, not SVG** for mermaid: mermaid-cli writes `width="100%"` on SVG, so `<img>` sizing is unpredictable.
- **PNG for draw.io exports too**: a draw.io SVG embeds its editable source (~500 KB) and carries `color-scheme: light dark`, which can tint in dark mode. A 2x PNG is smaller and stable.
- The table renderer strips backticks and emphasis; inline-code styling is lost but the text is intact.
- Verify before committing: `git status` shows the post + `.mmd` + `.png` + `tables/*.md`, and no mermaid fences or raw tables remain in the post.
- Commit with `-s` and a repo-style subject (`Add <topic> post`).
