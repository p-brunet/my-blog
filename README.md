# my-blog

Personal blog built with [MyST Markdown](https://mystmd.org/) and deployed to GitHub Pages. Topics: geospatial data, energy, data engineering, visualization.

## Structure

```
.
├── myst.yml                  # Site config (title, TOC, theme)
├── index.md                  # Landing page with post gallery
├── custom.css                # Custom styling
├── posts/                    # Blog posts (one .md per post)
│   └── images/               # Shared images
├── projects/                 # Category pages (data-engineering, maps, modelling)
├── generate_rss.py           # Generates rss.xml and atom.xml from post frontmatter
└── .github/workflows/
    ├── deploy.yml            # Build + deploy to GitHub Pages on push to main
    └── build.yml             # PR preview builds (Netlify)
```

## Adding a Post

1. Create `posts/my-post.md` with frontmatter: `title`, `date`, `authors`, `description`, `thumbnail`, `tags`, `keywords`
2. Add a card in the `## Posts` grid in `index.md`
3. Add a card in the matching `projects/*.md` category page

## Running Locally

```bash
pip install feedgen pyyaml
npm install -g mystmd
myst start
```

## Deployment

Pushing to `main` triggers `deploy.yml`, which builds the site, generates RSS/Atom feeds, and deploys to GitHub Pages.
