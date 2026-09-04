# desterhuizen.uk

Personal site for Dawid Esterhuizen, built with Jekyll and served by GitHub Pages
from `main` at [desterhuizen.uk](https://desterhuizen.uk) (`CNAME`).

The site is laid out as a man page: the homepage is `DESTERHUIZEN(7)`, with
`NAME`, `SYNOPSIS`, `DESCRIPTION`, `OPTIONS`, `REPOSITORIES`, `HISTORY`,
`CERTIFICATIONS`, `PUBLICATIONS`, `EXPERIMENTS` and `AUTHOR` sections.

## Layout

    _config.yml           site settings, permalinks, experiment_types
    index.html            homepage — the man page itself
    experiments.html      /experiments/ archive, redirects from /writing/
    _posts/               posts, one Markdown file per experiment
    _layouts/             default.html (shell, head, nav) and post.html
    _includes/            post-card.html, the link card used in both listings
    assets/css/style.css  the whole stylesheet, no framework
    assets/               favicons, webmanifest, post images

## Posts

Posts live in `_posts/` as `YYYY-MM-DD-slug.md` and publish to
`/experiments/:title/`. Front matter:

```yaml
---
layout: post
title:  "CLI Tools & Last Pass CLI"
date:   2021-02-02 20:00:00 +0100
categories: security
type:   article
redirect_from:
  - /writing/lpass-cli/
---
```

`type` groups the post on both the homepage and `/experiments/`, and must be one
of the values in `experiment_types` in `_config.yml` — currently `article`,
`writeup`, `exploit`. A post with any other `type` builds but is never listed.
The order of that list is the order the groups render in.

`redirect_from` is only needed for posts that already had a `/writing/` URL; it
is provided by `jekyll-redirect-from`.

## Local preview

```sh
bundle install
bundle exec jekyll serve
```

The `Gemfile` pins `github-pages` so a local build uses the same gem set GitHub
Pages builds with. That set needs Ruby 3.0 or newer — on the macOS system Ruby
2.6 it fails to resolve, because `ffi` 1.17 requires `>= 3.0`. A slimmer
local-only `Gemfile.verify` (git-ignored, vendored into `vendor/verify` via
`.bundle/config`) pins versions that do install there and builds the same site:

```sh
BUNDLE_GEMFILE=Gemfile.verify bundle exec jekyll build
```

## Deploying

Push to `main`. GitHub Pages builds and publishes it; there is no Actions
workflow. `_site/` is a local build artifact and is git-ignored.
