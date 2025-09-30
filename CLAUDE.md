# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is "The Last Dev Blog" - a Jekyll static site blog focused on building the tools that build the products. The site uses custom layouts and styles rather than a pre-built theme.

Jekyll is a Ruby-based static site generator that transforms Markdown files into a complete website.

## Development Commands

### Run the development server
```bash
bundle exec jekyll serve --baseurl ""
```
This starts a local server at http://localhost:4000 with auto-regeneration enabled. The `--baseurl ""` flag is required for local development since the site is configured for GitHub Pages deployment.

### Build the site without serving
```bash
bundle exec jekyll build
```

### Install dependencies
```bash
bundle install
```

## Project Structure

- `_posts/`: Blog posts in format `YYYY-MM-DD-title.md` or `.markdown`
- `_layouts/`: Custom layout templates (default.html, home.html, page.html, post.html)
- `_includes/`: Reusable HTML components
- `_site/`: Generated static site (do not edit directly)
- `assets/`: Static assets including images and custom CSS
  - `assets/css/main.scss`: Custom styling
  - `assets/images/`: Blog post images and diagrams
- `_config.yml`: Main Jekyll configuration
- Favicon files: Multiple sizes (16x16, 32x32, 96x96, SVG) and platform-specific icons
- Posts require front matter with `layout`, `title`, `date`, and `categories`

## Key Configuration

- Site title: The Last Dev Blog
- Custom layouts (not using minima theme)
- Jekyll version: 4.4.1
- Plugins: jekyll-feed
- Social links configured: Twitter (@jrodriguexg), GitHub (JoseFdri), LinkedIn (jose-rodriguez-g)

## Content

Current blog posts focus on:
- Agentic coding and AI-assisted development workflows