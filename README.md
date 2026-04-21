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

## Auto sync and reload after push

This repository includes a workflow:

- `.github/workflows/content-sync-and-reload.yml`

What it does on push to `main` (for `content/blog/**` and `content/notes/**`):

1. SSH to your server and sync `/opt/aether_garden/aether_garden.content` to the pushed commit SHA.
2. Call website backend endpoint `POST /internal/content/reload`.

Required repository secrets:

- `DEPLOY_HOST`
- `DEPLOY_USER`
- `DEPLOY_SSH_KEY`
- `DEPLOY_PORT` (optional, default `22`)
- `CONTENT_REPO_DIR` (optional, default `/opt/aether_garden/aether_garden.content`)
- `WEBSITE_RELOAD_URL` (for example `https://duckran.top/internal/content/reload`)
- `WEBSITE_RELOAD_TOKEN` (must match backend `InternalAuth:ReloadToken`)
