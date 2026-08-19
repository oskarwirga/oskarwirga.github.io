# oskarwirga.github.io

A bare-bones Jekyll blog for GitHub Pages. Markdown files in `_posts` become
published posts; everything else is intentionally minimal.

## Write

Create a Markdown file in `_posts` named `YYYY-MM-DD-title.md`:

```md
---
title: "Post title"
description: "Optional short description."
---

Start writing here.
```

Keep unfinished work outside `_posts`. The `_drafts` folder is available for
local drafts, but files committed there will still be visible in the public
GitHub repository even though Jekyll does not publish them.

## Preview locally

Install Ruby and Bundler, then run:

```sh
bundle install
bundle exec jekyll serve --livereload
```

Open `http://localhost:4000`.

## Publish

Push to `main`. The included GitHub Pages workflow builds and publishes the
site. In the repository settings, set **Pages → Build and deployment → Source**
to **GitHub Actions** if it is not already selected.
