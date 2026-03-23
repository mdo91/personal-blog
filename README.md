# Mahmut Aoata Blog

Personal blog built with [Astro](https://astro.build/) using a customized version of the AstroPaper starter.

## What this repo is

- Personal website and blog for **Mahmut Aoata**
- Content-driven setup using Markdown posts
- Clean, minimal UI with fast static pages

## Quick start

```bash
npm install
npm run dev
```

Then open:

- `http://localhost:4321/` (or the next free port Astro shows)

## Where to edit things

- **Site settings:** `src/config.ts`
  - title, author, description, social links
- **Blog posts:** `src/content/blog/`
  - create a `.md` file with frontmatter + markdown body
- **Theme styles:** `src/styles/base.css`
- **Public assets:** `public/`

## Post format

Create files in `src/content/blog/` like:

```md
---
title: My Post Title
description: One sentence summary.
pubDatetime: 2026-03-23T13:15:00Z
tags:
  - notes
  - dev
---

# Heading

Write your content in markdown.
```

## Useful commands

| Command | What it does |
| --- | --- |
| `npm run dev` | Run local dev server |
| `npm run build` | Build static site into `dist/` |
| `npm run preview` | Preview built site |
| `npm run lint` | Run lint checks |
| `npm run format` | Format files with Prettier |

## Generated output

- Build output is generated in `dist/`
- Generated blog pages are in `dist/posts/` after `npm run build`

## Deploy notes

This project can be deployed to any static host (Cloudflare Pages, Netlify, Vercel, GitHub Pages, etc.).

- Current production URL: `https://mdo91.github.io/personal-blog/`
- About page in production: `https://mdo91.github.io/personal-blog/about/`
- Note: `https://mdo91.github.io/about/` is expected to 404 until a custom/root domain setup is added.

## Credits

- Original theme base: [pages-cms/astro-blog-template](https://github.com/pages-cms/astro-blog-template)
