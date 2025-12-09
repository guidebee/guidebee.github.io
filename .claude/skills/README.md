# Claude Code Skills

This directory contains custom skills for Claude Code in the Solana Trading System blog repository.

## Available Skills

### blog-writer

**Purpose**: Write technical blog posts documenting Solana Trading System development progress.

**Usage**: Simply invoke the skill when you want to write a blog post:

```
Use the blog-writer skill to write a post about today's work on the executor service
```

Or more simply:

```
Write a blog post about the NATS event publishing work
```

**What it does**:
- Automatically reads recent posts from `_posts` to understand format and context
- Gathers technical details from `C:\Trading\solana-trading-system`
- Writes comprehensive posts following the established format
- Includes code examples, architecture diagrams, and monitoring details
- Links to related posts and documentation
- No questions needed - has all the context!

**Context Locations**:
- Blog posts: `C:\Trading\guidebee.github.io\_posts`
- Trading system code: `C:\Trading\solana-trading-system`
- Images: `C:\Trading\guidebee.github.io\posts\{YYYY}\{MM}\images`

## How Skills Work

Claude Code automatically discovers skills by reading `.md` files in this directory. When you mention topics related to a skill, Claude will use that skill's guidelines and context.

## Adding New Skills

Create a new `.md` file in this directory with:
1. Clear title and purpose
2. Context about repositories and file locations
3. Format/structure guidelines
4. Examples of usage
5. Any domain-specific knowledge

Skills are automatically available once the file is created - no configuration needed!
