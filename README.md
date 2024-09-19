# MasonHugoBlog
[![Deploy Hugo to GitHub Pages](https://github.com/MasonCodingHere/MasonHugoBlog/actions/workflows/deploy.yml/badge.svg?branch=main&event=push)](https://github.com/MasonCodingHere/MasonHugoBlog/actions/workflows/deploy.yml)
This repository is the Hugo blog source code.

## Install Hugo
[Install Hugo](https://gohugo.io/installation/)
## Install a Theme
Pick a theme from [Hugo Themes](https://themes.gohugo.io/)

I picked *stack* as my bolg theme and adopted Git submodule installation.

[hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack)

## Config Blog
I think the example site from stack theme is sufficient for me, so I copied `/themes/hugo-theme-stack/exampleSite/content/` to `/content` of my blog root directory.

Delete default `hugo.toml` in root directory, and copied `/themes/hugo-theme-stack/exampleSite/hugo.yaml/` to the root directory as my config file.

I copied `/themes/hugo-theme-stack/archetypes/` to `/archetypes` of my blog root directory.

make a directory named `img` in `/assets` and put a photo named `avatar.png` as blog avatar.

## Push to GitHub

Push it to a GitHub repo.


## Set GitHub Action

1. create a new repo named `username.github.io`.

2. set deploy key for two repos.

3. write a `./.github/workflows/deploy.yml` in first repo.

