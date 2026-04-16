# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 🚀 Quick Start: Activate Blog Writer Skill

**IMPORTANT**: When starting a Claude Code session in this repository, immediately activate the blog-writer skill:

```
Use the blog-writer skill
```

This skill provides complete context for writing technical blog posts about Solana Trading System development, including:
- Access to recent posts in `_posts/` for format and context
- Knowledge of the trading system repo at `C:\Trading\solana-trading-system`
- Automatic formatting following Jekyll conventions
- No need to specify formats or provide instructions

Once activated, you can simply say "write a blog post about [topic]" and the skill handles everything.

## Repository Overview

This is a Jekyll-based GitHub Pages blog documenting the development of a Solana Trading System. The blog uses the Academic Pages template and is deployed at https://guidebee.github.io.

## Running the Site Locally

### Using Jekyll (Recommended for Development)

```bash
# Install dependencies (first time only)
bundle install

# If you get permission errors, install gems locally:
bundle config set --local path 'vendor/bundle'
bundle install

# Run the development server
jekyll serve -l -H localhost

# Or use bundle exec to ensure correct dependencies
bundle exec jekyll serve -l -H localhost
```

The site will be available at `http://localhost:4000`. Jekyll auto-reloads on Markdown and HTML changes, but requires restart for `_config.yml` changes.

### Using Docker

```bash
chmod -R 777 .
docker compose up
```

Site available at `http://localhost:4000`.

### Using VS Code DevContainer

Press **F1** → **DevContainer: Reopen in Container** to automatically start the site in a container.

## Blog Post Structure

### File Naming and Location

- **Location**: `_posts/`
- **Naming**: `YYYY-MM-DD-url-slug.md`
- **Images**: `posts/{YYYY}/{MM}/images/` with relative references `../images/filename.png`

### Jekyll Front Matter Template

```yaml
---
layout: single
title: "Descriptive Title"
date: YYYY-MM-DD
permalink: /posts/YYYY/MM/url-slug/
categories:
  - blog
tags:
  - solana
  - trading
  - [other tags]
excerpt: "Brief summary for previews and SEO."
---
```

### Content Structure

Blog posts follow this pattern:

1. **TL;DR section** - 2-4 bullet points of key accomplishments
2. **Technical sections** with clear headers (##, ###)
3. **Code examples** in fenced blocks with language tags
4. **Architecture diagrams** using ASCII/text diagrams
5. **Impact and Next Steps** - benefits and future work
6. **Conclusion** - brief wrap-up
7. **Related Posts** - links to other posts in series
8. **Technical Documentation** - links to GitHub docs
9. **Footer** - Connect section with GitHub/LinkedIn, series note

### Post Numbering

Posts are numbered sequentially in the series footer:
```markdown
*This is post #X in the Solana Trading System development series...*
```

Check existing posts in `_posts/` to determine the next number.

## Related Repository: Solana Trading System

The blog documents work from the trading system repository at `C:\Trading\solana-trading-system`. When writing blog posts:

- Read relevant README.md files from service directories
- Reference code architecture and implementation details
- Include monitoring/observability information
- Link to GitHub documentation in posts

## Site Configuration

### Key Settings (_config.yml)

- **Site title**: "James Shen - Solana Trading System Development"
- **Author**: James Shen (guidebee)
- **Markdown**: kramdown with GFM input
- **Permalink structure**: `/:categories/:title/`
- **Collections**: portfolio (output: true), others (output: false)

### Jekyll Plugins

- jekyll-feed
- jekyll-sitemap
- jekyll-redirect-from
- jemoji

## Content Collections

- `_posts/` - Blog posts (main content)
- `_pages/` - Static pages (About, CV, etc.)
- `_portfolio/` - Portfolio items (output enabled)
- `_publications/` - Publications (output disabled)
- `_talks/` - Talks (output disabled)
- `_teaching/` - Teaching materials (output disabled)

## Custom Skill: blog-writer

A custom Claude Code skill exists at `.claude/skills/blog-writer.md` that automatically:

1. Reads recent posts from `_posts/` for context and format
2. Gathers technical details from `C:\Trading\solana-trading-system`
3. Follows the established blog post format
4. Includes code examples, architecture diagrams, monitoring sections
5. Links to related posts and GitHub documentation

**To use**: Simply ask Claude to "write a blog post about [topic]" - no format specification needed.

## Important Paths

- **Blog posts**: `_posts/`
- **Static pages**: `_pages/`
- **Images**: `posts/{YYYY}/{MM}/images/` or `images/`
- **Site config**: `_config.yml`
- **Local settings**: `.claude/settings.local.json`
- **Custom skills**: `.claude/skills/`

## Jekyll Behavior Notes

- Changes to `_config.yml` require server restart
- Markdown (*.md) and HTML changes auto-reload with `-l` flag
- Site builds to `_site/` (excluded from git)
- Local gems install to `vendor/bundle/` (excluded from git)

## Documentation Style

When documenting Solana Trading System work:

- **Technical depth**: Detailed enough for developers to understand
- **Code examples**: Always include practical examples with proper syntax highlighting
- **Architecture**: Use text/ASCII diagrams for system flow
- **Performance**: Include metrics, benchmarks, latency characteristics
- **Monitoring**: Document Prometheus metrics, Grafana dashboards, OpenTelemetry traces
- **Configuration**: Show command-line flags, environment variables, config files
- **Integration**: Explain how components interact (quote-service → NATS → scanner-service)

## Common Tags

- solana, trading, infrastructure, nats, market-data, observability, monitoring, typescript, go, rust, architecture, performance, oracle, dex, arbitrage, refactoring, testing
