# Project Overview

This is a Jekyll-based static site blog. The blog's content focuses on software engineering best practices, with a particular emphasis on building scalable and maintainable projects. The author's posts advocate for modularization, comprehensive testing, and the use of abstractions to reduce code duplication.

The site is configured for deployment on GitHub Pages.

# Building and Running

## Prerequisites

*   Ruby (with Bundler)
*   Jekyll

## Install Dependencies

```bash
bundle install
```

## Run Development Server

```bash
bundle exec jekyll serve --baseurl ""
```

The site will be available at `http://localhost:4000`.

## Build Site

```bash
bundle exec jekyll build
```

The generated static site will be in the `_site/` directory.

# Development Conventions

## Writing Posts

New posts are created in the `_posts/` directory with the following naming convention: `YYYY-MM-DD-title.md`.

All posts must include the following front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +/-TTTT
categories: category-name
---
```

## Coding Style

The existing blog posts and code examples suggest a preference for:

*   **Modular Code:** Breaking down large files into smaller, more focused modules.
*   **Clear Naming:** Using descriptive names for files, functions, and variables.
*   **In-depth Explanations:** Providing detailed explanations and code examples to illustrate concepts.

## Architectural Patterns

The blog posts highlight the importance of the following architectural patterns:

*   **Service Layer:** Separating database interactions from business logic.
*   **Configuration-driven UI:** Using configuration objects to generate UI components, such as forms.
*   **Testing:** Writing unit and integration tests to ensure code quality.
*   **Automation:** Using linters, Git hooks, and CI/CD pipelines to automate quality checks.
