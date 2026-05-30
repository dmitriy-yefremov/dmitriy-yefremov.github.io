# yefremov.net

Source for [yefremov.net](https://yefremov.net) — a personal blog about software
development, built with [Jekyll](https://jekyllrb.com) 4 and hosted on GitHub Pages.

## Local development

Requires Ruby (3.1+) and Bundler.

```sh
bundle install
bundle exec jekyll serve
```

The site is served at <http://localhost:4000> with live reload.

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md` with front matter:

```yaml
---
layout: post
title: My post title
---
```

Use `<!--more-->` to mark where the excerpt ends on the home page.

## Deployment

Pushes to `master` are built and deployed automatically by the GitHub Actions
workflow in [`.github/workflows/jekyll.yml`](.github/workflows/jekyll.yml).
In the repo settings, **Settings → Pages → Build and deployment → Source** must be
set to **GitHub Actions**.

## Comments

Comments use [giscus](https://giscus.app) (backed by GitHub Discussions). To enable:

1. Enable **Discussions** on the repository.
2. Install the [giscus app](https://github.com/apps/giscus) on the repo.
3. Run the configurator at <https://giscus.app> and copy the generated
   `repo`, `repo_id`, `category`, and `category_id` values into the `giscus`
   section of [`_config.yml`](_config.yml).

Until those values are filled in, the comments section simply doesn't render.
