# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll static site blog using the Minima theme. Jekyll is a Ruby-based static site generator that transforms Markdown files into a complete website.

## Development Commands

### Run the development server
```bash
bundle exec jekyll serve
```
This starts a local server at http://localhost:4000 with auto-regeneration enabled.

### Build the site without serving
```bash
bundle exec jekyll build
```

### Install dependencies
```bash
bundle install
```

## Project Structure

- `_posts/`: Blog posts in format `YYYY-MM-DD-title.markdown`
- `_site/`: Generated static site (do not edit directly)
- `_config.yml`: Main Jekyll configuration
- Posts require front matter with `layout`, `title`, `date`, and `categories`

## Key Configuration

- Theme: minima 2.5
- Jekyll version: 4.4.1
- Plugins: jekyll-feed