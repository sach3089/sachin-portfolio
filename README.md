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

Publish the contents of `dist/` to any static host. The generated site uses root-relative paths, so deploy at the domain root, rather than under a repository subpath. The local server supports directory URLs and a custom 404 page.

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

Build with `npm run build` and deploy `dist/` to a static host. No application server or database is required. This repository includes the full source and static output, including the résumé and social assets.

The site uses root-relative URLs. Host at a domain root (custom domain or user-level Pages site), or configure your host to serve this folder at the root. Set `SITE_URL` to the final origin when building so canonical URLs, RSS, and sitemap use the right domain.

No live hosted domain has been confirmed yet. The original Sites project metadata is local-only and is excluded from this GitHub repository. The repository is portable to any static hosting service.
