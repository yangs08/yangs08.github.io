# yangs08.github.io

Personal Jekyll blog for writing about AI Agent, RAG, and engineering notes.

## Quick Start

```bash
bundle install
bundle exec jekyll serve
```

Open http://127.0.0.1:4000 in your browser.

## Homepage

The homepage uses a compact personal-blog layout inspired by awen.me:

- profile intro
- article search entry linking to the archive
- latest blog posts
- Projects section linking to https://github.com/yangs08

## Writing a Post

Create a new `.md` file in `_posts/` with the naming convention `YYYY-MM-DD-title.md`:

```markdown
---
layout: post
title: "Your Post Title"
date: 2026-05-18
---

Content here...
```

## Analytics

Analytics configuration lives in `_config.yml`:

- `baidu.id` enables Baidu Analytics through `hm.baidu.com`.
- `ga.id` enables Google Analytics 4 through `gtag.js`.
- Busuanzi page-view counting is loaded in `_includes/head.html` and displayed in `_includes/footer.html`.

Leave an analytics ID blank to disable that provider.

## Directory Structure

```
.
├── _posts/          # Blog posts (Markdown)
├── _includes/       # Reusable HTML components
├── _layouts/        # Page layout templates
├── _config.yml      # Site configuration
├── css/             # Stylesheets
├── images/          # Images
└── js/              # JavaScript
```

## License

This project is for personal use. Original theme by [leopardpan](https://github.com/leopardpan/leopardpan.github.io).
