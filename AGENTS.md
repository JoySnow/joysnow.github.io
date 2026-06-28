# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is this?

A personal blog built with **Jekyll** (static site generator), using the **NexT** theme, hosted on **GitHub Pages** at `joysnow.github.io`.

## Development

```bash
# Install dependencies and serve locally
# (Gemfile needed for local build with remote_theme)
echo "source 'https://rubygems.org'
gem 'jekyll-remote-theme'
gem 'github-pages', group: :jekyll_plugins" > Gemfile
bundle install
bundle exec jekyll serve --livereload
```

## Deploy

Push to `master` branch — GitHub Pages auto-builds and deploys.

## Creating a New Post

Add a Markdown file to `_posts/` with naming convention `YYYY-MM-DD-title.md`:

```yaml
---
layout: post
title: "Your Post Title"
tags: [tag1, tag2]
categories: [category-name]
---
```

## Architecture

| Path | Purpose |
|---|---|
| `_config.yml` | Site config + NexT theme settings |
| `_posts/*.md` | Blog posts |
| `archives/index.html` | Archive page (layout: archive) |
| `tags/index.html` | Tag cloud page (layout: page, type: tags) |
| `tag/index.html` | Single tag filter page (layout: tag) |
| `categories/index.html` | Category listing page (layout: page, type: categories) |
| `category/index.html` | Single category filter page (layout: category) |
| `about.md` | About page |

Theme is pulled from `Simpleyyt/jekyll-theme-next` via `remote_theme` at build time. No theme code is stored in this repo.
