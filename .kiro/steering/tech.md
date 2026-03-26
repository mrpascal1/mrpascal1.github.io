# Tech Stack

## Framework
- **Jekyll** static site generator (via GitHub Pages)
- Uses `remote_theme: ankitsultana/researcher@gem` — the "Researcher" theme
- Markdown processor: **kramdown**

## Styling
- **SCSS** compiled via Jekyll's built-in Sass pipeline
- Entry point: `css/main.scss` → imports from `_sass/`
- Key files:
  - `_sass/vars.scss` — accent color (`#FF0F00`), layout widths
  - `_sass/_style.scss` — base styles, navbar, responsive breakpoints
  - `_sass/typography.scss`, `_sass/tables.scss` — typography and table styles
  - `_sass/rouge-github.scss` — syntax highlighting

## Analytics
- Google Analytics via `tracking_id` in `_config.yml`

## Layouts
- `_layouts/default.html` — base layout for pages
- `_layouts/post.html` — layout for blog posts

## Common Commands

```bash
# Serve locally (requires Ruby + Jekyll installed)
bundle exec jekyll serve

# Build static site
bundle exec jekyll build
```

> Note: The site uses `remote_theme`, so local builds require the theme gem to be available. GitHub Pages handles deployment automatically on push to the default branch.
