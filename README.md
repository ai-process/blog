# KTMS Dev Blog

[![Deploy GitHub Pages](https://github.com/ai-process/blog/actions/workflows/deploy.yml/badge.svg)](https://github.com/ai-process/blog/actions/workflows/deploy.yml)

The official engineering blog for [**KTMS**](https://github.com/ai-process/ktms), [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter), and the AI Process infrastructure ecosystem.

Live site: [**https://blog.ai-process.dev**](https://blog.ai-process.dev/)

## Publication Cadence & Policy

- **Cadence:** We publish a new post every **2 days** whenever there is news, releases, or architectural insights.
- **Tone:** Practical, educational, and grounded in real-world production engineering.
- **Standard format:** Each post addresses a core problem or concept (abstract overview) followed by concrete implementation patterns and references to our open-source tools.

## Writing a New Post

Create a new markdown file inside `_posts/` with the filename format:

```
_posts/YYYY-MM-DD-title-slug.md
```

Front matter structure:

```yaml
---
layout: post
title: "Title of the Post"
date: YYYY-MM-DD HH:MM:SS +0000
reading_time: 5
tags:
  - architecture
  - ktms
  - tokenmeter
description: "Brief summary of the article."
---

Your content here...
```

## Local Development

If you have Ruby and Bundler installed:

```bash
bundle install
bundle exec jekyll serve
```

Access the preview at `http://localhost:4000/blog/`.

## Deployment

The blog is automatically built and deployed to GitHub Pages via GitHub Actions whenever changes are pushed to the `main` branch (see [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)).
