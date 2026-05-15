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
├── tags/                     # Tag pages for sidebar navigation
├── generate_rss.py           # Generates rss.xml and atom.xml from post frontmatter
├── inject_comments.py        # Injects Giscus comment widget into built HTML
└── .github/workflows/
    ├── deploy.yml            # Build + deploy to GitHub Pages on push to main
    └── build.yml             # PR preview builds (Netlify)
```

## Adding a Post

1. Create `posts/my-post.md` with frontmatter: `title`, `date`, `authors`, `description`, `thumbnail`, `tags`, `keywords`
2. Add a card in the `## Posts` grid in `index.md`
3. If using a new tag, create `tags/my-tag.md` and add it to `project.toc` in `myst.yml`

## Running Locally

```bash
pip install -r requirements.txt
npm install -g mystmd
myst start
```

## Deployment

Pushing to `main` triggers `deploy.yml`, which builds the site, generates RSS/Atom feeds, injects Giscus comments, and deploys to GitHub Pages.
