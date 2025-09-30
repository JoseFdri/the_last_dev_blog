# The Last Dev Blog

A Jekyll-based static site blog focused on building the tools that build the products.

## Prerequisites

- Ruby (with Bundler)
- Jekyll 4.4.1

## Getting Started

### Install Dependencies

```bash
bundle install
```

### Run Development Server

```bash
bundle exec jekyll serve
```

The site will be available at `http://localhost:4000` with auto-regeneration enabled.

### Build Site

```bash
bundle exec jekyll build
```

The generated static site will be in the `_site/` directory.

## Project Structure

```
.
├── _posts/          # Blog posts (YYYY-MM-DD-title.md format)
├── _layouts/        # Custom layout templates
├── _includes/       # Reusable HTML components
├── assets/          # Static assets
│   ├── css/        # Custom styling
│   └── images/     # Blog images and diagrams
├── _config.yml      # Jekyll configuration
└── _site/          # Generated site (do not edit)
```

## Writing Posts

Create new posts in `_posts/` with the naming format: `YYYY-MM-DD-title.md`

Required front matter:
```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +/-TTTT
categories: category-name
---
```

## Configuration

- Site uses custom layouts (no pre-built theme)
- Plugins: jekyll-feed
- Social links: Twitter, GitHub, LinkedIn

## Contact

- Twitter: [@jrodriguexg](https://twitter.com/jrodriguexg)
- GitHub: [JoseFdri](https://github.com/JoseFdri)
- LinkedIn: [jose-rodriguez-g](https://linkedin.com/in/jose-rodriguez-g)