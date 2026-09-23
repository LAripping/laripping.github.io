# AGENTS.md

## Repository overview

- This repository publishes the GitHub Pages site at `laripping.github.io`, which redirects to `https://laripping.com`.
- Canonical remote: `https://github.com/LAripping/laripping.github.io`
- Primary working branch: `hydeout`.
- The site is a Jekyll blog using the Hydeout theme, with source content and theme customizations checked into this repository.

## Project layout

- `_posts/` — dated blog posts in Markdown with YAML front matter.
- `_layouts/` and `_includes/` — Liquid templates and reusable partials.
- `_sass/` and `assets/` — Sass source and static site assets.
- `_config.yml` — Jekyll site configuration and enabled plugins.
- `category/` — category landing pages.

## Local development

Choose Ruby 3.1.4, use a Docker container to build & run within.

1. First create a volume if not existing

```
$ docker volume create laripping-node-modules
```

2. Then build and serve

```
$ docker run -it -v "$PWD:/work" -v laripping-bundle:/user/local/bundle -p 4000:4000  -w /work ruby:3.1.4-bookworm bash
$ docker run -it -v "$PWD:/work" -v laripping-bundle:/user/local/bundle -w /work ruby:3.1.4-bookworm bash -lc 'bundl
e config set path /user/local/bundle && bundle install && bundle exec jekyll build && bundle exec jekyll serve --livereload --host 0.0.0.0'
```

Generated output lives in `_site/` and must not be committed.

## Styles

Sass is linted through npm:

```text
npm install
npm run stylelint
```

Keep Sass changes inside `_sass/` consistent with the existing Stylelint configuration.

## Content and template guidance

- Preserve Jekyll/Liquid syntax and valid YAML front matter when editing Markdown, layouts, or includes.
- Retain existing permalink and URL conventions unless a URL migration is explicitly requested.
- Add post images and other static resources under `assets/` or alongside the relevant `from-labs/` content, following nearby examples.
- Check desktop and narrow-screen rendering when modifying layouts, navigation, or Sass.
- For public-facing metadata changes, verify the home page, a standard page, and an article's social-share previews where relevant.

## Before finishing changes

- Run the narrowest relevant validation: `bundle exec jekyll build` for site/content/template changes and `npm run stylelint` for Sass changes.
- Do not commit generated files, dependency directories, caches, or local configuration ignored by `.gitignore`.
- Keep changes focused; do not reformat unrelated Markdown, Liquid, or Sass.
