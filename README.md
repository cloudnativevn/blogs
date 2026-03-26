# Cloud Native Vietnam

Community blog for [Cloud Native Vietnam](https://github.com/cloudnativevn) — sharing knowledge about Kubernetes, cloud native technologies, and the Vietnamese tech ecosystem.

## Local Development

### Prerequisites

- Ruby 4.x
- Bundler

### Setup

```bash
bundle install
bundle exec jekyll serve
```

Visit `http://localhost:4000` to preview the site.

### Writing a New Post

1. Create a new file in `_articles/` (e.g. `my-post-title.md`)
2. Add front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: 2026-03-24
author: Your Name
tags: [tag1, tag2]
---
```

3. Write your content in Markdown
4. Submit a pull request

## Deployment

The site is automatically deployed to GitHub Pages via GitHub Actions when changes are pushed to the `main` branch.

## License

[MIT](LICENSE)
