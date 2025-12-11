# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based static site hosted on GitHub Pages. The site uses the Minima theme and runs on Jekyll 3.5.1. Content is primarily blog posts stored in the `_posts` directory written in Markdown.

## Development Commands

### Local Development Server
```bash
bundle exec jekyll serve
```
This starts a local development server on port 5000 (configured in `_config.yml`).

**Important:** The `_config.yml` file is NOT automatically reloaded when using `jekyll serve`. You must restart the server after changing this file.

### Build Site
```bash
bundle exec jekyll build
```
Generates the static site in the `_site` directory.

### Install Dependencies
```bash
bundle install
```
Installs Ruby gems specified in the Gemfile.

## Architecture

### Directory Structure
- `_posts/` - Blog posts in Markdown format with YAML front matter. Files must follow naming convention: `YYYY-MM-DD-title.markdown`
- `_site/` - Generated static site (ignored by git)
- `_config.yml` - Jekyll configuration file
- `index.md` - Homepage (uses the "home" layout from Minima theme)
- `about.md` - About page
- `Gemfile` - Ruby dependencies

### Content Creation

Blog posts go in `_posts/` with the following front matter structure:
```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
categories: jekyll update
---
```

### Theme
Uses the Minima theme (version ~> 2.0). To customize layouts, you would need to override theme defaults by creating corresponding files in `_layouts/` directory.

## Jekyll Configuration

- **Port:** Custom port 5000 (instead of default 4000)
- **Markdown processor:** kramdown
- **Plugins:** jekyll-feed

## Git Ignored Files
- `_site/` - Build output
- `.sass-cache/` - Sass compilation cache
- `.jekyll-metadata` - Jekyll metadata cache
