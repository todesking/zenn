# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Zenn content repository for publishing technical articles and books on the Japanese technical publishing platform Zenn (zenn.dev). The repository uses `zenn-cli` to manage and preview content locally.

## Common Commands

### Content Management
- `npx zenn preview` - Start local preview server (opens browser at http://localhost:8000)
- `npx zenn new:article` - Create a new article with auto-generated slug
- `npx zenn new:article --slug my-article-slug` - Create article with custom slug
- `npx zenn new:book` - Create a new book
- `npx zenn list:articles` - List all articles
- `npx zenn list:books` - List all books

### Development Workflow
1. Create new content using the commands above
2. Edit markdown files in `/articles/` or `/books/`
3. Preview changes with `npx zenn preview`
4. Commit changes when satisfied
5. Push to GitHub to automatically publish on Zenn

## Repository Structure

- `/articles/` - Individual technical articles in markdown format
  - Each article has YAML frontmatter with metadata (title, emoji, type, topics, published)
- `/books/` - Book content organized in subdirectories
- `package.json` - Node.js configuration with zenn-cli dependency

## Content Guidelines

### Article Frontmatter Format
```yaml
---
title: "Article Title"
emoji: "🎉"
type: "tech" # tech or idea
topics: ["topic1", "topic2"] # up to 5 topics
published: true # or false for drafts
---
```

### Zenn Markdown Extensions
- Supports standard markdown
- Code blocks with syntax highlighting
- Math expressions with KaTeX
- Message boxes: :::message, :::message alert
- Details/Accordion: :::details
- Embeds: @[youtube], @[tweet], @[codepen], etc.

## Important Notes

- This repository may be running in a sandboxed environment (see articles/claude-code-with-sandbox-exec.md)
- File writes may be restricted to the project directory only
- No testing framework is configured - focus on content creation
- The repository is connected to GitHub for automatic publishing to Zenn

## Writing Style Guidelines

- 文体は「だ、である」調にする