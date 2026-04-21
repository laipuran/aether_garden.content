# aether-content-example

Example markdown content repository for `aether_garden-be`.

## Structure

- `content/blog`: blog posts
- `content/notes`: short notes

## Front matter requirements

Each markdown file should start with YAML front matter:

```md
---
slug: my-first-post
title: My First Post
date: 2026-04-21
excerpt: Optional summary text
tags:
  - demo
  - sample
status: published
updatedAt: 2026-04-21
---

Body content here.
```

`slug`, `title`, and `date` are required.
`status` defaults to `published`; set it to another value to hide drafts.
