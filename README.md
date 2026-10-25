# Sachin Chauhan — Portfolio & Engineering Notebook

A fully static portfolio based on the approved Midnight Engineering design. No database, server runtime, analytics, or third-party browser dependencies.

## Local preview

Requires Node.js 20 or newer. No dependency installation is needed.

```sh
npm run dev
```

Open http://127.0.0.1:4173. Restart after editing files to rebuild.

## Build

```sh
npm run build
```

Publish the contents of `dist/` to any static host. The generator supports both domain-root and repository-subpath hosting using SITE_URL. The local server supports directory URLs and a custom 404 page.

For a different domain, set its origin before building so canonical URLs, RSS, sitemap, and sharing metadata use the correct address:

```sh
SITE_URL=https://your-domain.example npm run build
```

## Publish an article

Create a file such as `content/posts/my-first-article.md`:

```markdown
---
title: My first article
description: A short summary for the article listing and search previews.
date: 2026-10-25
tags: Java, AWS
draft: false
---
## The problem

Your article starts here.

- Explain what you learned.
- Include sanitized examples.
```

The filename becomes the URL: `/blog/my-first-article/`.

Rebuild and redeploy to publish. The generator automatically adds published articles to the notebook, homepage, search, RSS feed, and sitemap. `draft: true` excludes a file from all generated pages. A starter draft is provided; no articles are published by default. Use real publication dates and review workplace details before publishing.

Supported Markdown: paragraphs, headings, unordered lists, fenced code blocks, inline code, bold text, and HTTPS/HTTP links. Raw HTML is escaped. Images, tables, nested lists, blockquotes, and MDX are not supported by this deliberately small parser.

## Edit portfolio content

- `src/home.html`: profile, experience, project story, and contact section.
- `src/style.css`: responsive layout and dark/light themes.
- `src/app.js`: architecture explorer, project disclosure, theme toggle, article search, and code-copy controls.
- `build.mjs`: shared header/footer, article templates, metadata, RSS, sitemap, and static build.
- `public/resume.pdf`: current downloadable résumé.
- `public/social-card.png`: sharing image.

## Hosting

GitHub Pages deployment is configured in `.github/workflows/pages.yml`. A push to `main` builds the Markdown content and publishes `dist/` automatically. No database, custom server, or deployment credentials are required.

Hosting address: https://sach3089.github.io/sachin-portfolio/

For a custom domain, configure it in GitHub Pages settings, add the DNS records supplied by GitHub, and update the workflow's SITE_URL. The generator adjusts navigation, assets, canonical URLs, RSS, and sitemap for the origin and optional path in SITE_URL.

Local development still uses npm run dev. Original Sites metadata is local-only and excluded from GitHub.
