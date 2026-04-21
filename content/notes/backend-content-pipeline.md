---
slug: backend-content-pipeline
title: Backend Content Pipeline
date: 2026-04-21
excerpt: Quick note about how markdown files are loaded.
tags:
  - backend
  - notes
status: published
updatedAt: 2026-04-21
---

The backend scans markdown files in configured directories.

It parses YAML front matter, validates required fields, and builds API responses from the parsed result.
