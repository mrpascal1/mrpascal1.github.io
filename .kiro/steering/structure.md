# Project Structure

```
/
├── _config.yml          # Site config: title, nav links, theme, analytics, footer
├── index.md             # Homepage (layout: default)
├── about.md             # About page (layout: default)
├── blogs.md             # Blog listing page — iterates site.posts
│
├── _posts/              # Blog posts — filename format: YYYY-MM-DD-slug.md
├── _layouts/
│   ├── default.html     # Base page layout
│   └── post.html        # Blog post layout
│
├── _sass/               # SCSS partials
│   ├── vars.scss        # Color and width variables
│   ├── _style.scss      # Main styles (imports vars, rouge-github)
│   ├── typography.scss
│   ├── tables.scss
│   └── rouge-github.scss
│
├── css/
│   └── main.scss        # SCSS entry point (Jekyll processes this)
│
├── imgs/                # Images organized by YYYYMM subdirectory
├── flockr/              # App-specific pages (privacy policy, T&C)
└── shahid_raza_resume.pdf
```

## Conventions

### Blog Posts
- Filename: `YYYY-MM-DD-kebab-case-title.md` inside `_posts/`
- Required front matter:
  ```yaml
  ---
  layout: post
  author: Shahid Raza
  title: Post Title Here
  tags: Tag1 Tag2 Tag3
  ---
  ```
- Use `<!--more-->` after the opening paragraph to define the excerpt shown on the blog listing
- Optional: `math: true` for posts using MathJax (LaTeX math rendering)

### Pages
- Root-level `.md` files with `layout: default` front matter
- Navigation links are configured in `_config.yml` under `nav:`

### Images
- Store in `imgs/YYYYMM/` subdirectory matching the post date
- Reference with relative paths in markdown

### Styling
- Customize accent color in `_sass/vars.scss` (`$accent`)
- Max content width is `700px` (`$max-width` in vars.scss)
- Responsive breakpoints: 800px, 650px, 500px
