# Ishan Joshi's Blog

A personal blog built with Jekyll and hosted on GitHub Pages.

## Getting Started

### Prerequisites

- Ruby (version 2.5 or higher)
- Bundler gem

### Installation

1. Install dependencies:
```bash
bundle install
```

2. Build and serve the site locally:
```bash
bundle exec jekyll serve
```

3. Open your browser and navigate to `http://localhost:4000`

## Creating New Posts

Create new blog posts in the `_posts` directory. Files should be named with the format:

```
YEAR-MONTH-DAY-title.md
```

For example: `2024-01-15-my-first-post.md`

Each post should start with front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: 2024-01-15 10:00:00 -0500
categories: category1 category2
---
```

## Customization

- Edit `_config.yml` to customize site settings
- Modify `index.html` to change the homepage
- Update `about.md` with your information
- Customize the theme by editing CSS in `assets/css/` or overriding theme files

## Deployment

This site is automatically deployed to GitHub Pages when you push to the `main` branch. GitHub Pages will automatically build and serve your Jekyll site.

## Resources

- [Jekyll Documentation](https://jekyllrb.com/docs/)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [Minima Theme](https://github.com/jekyll/minima)

