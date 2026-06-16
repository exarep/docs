# Exarep documentation site

Documentation for the Exarep project, built with [Material for MkDocs](https://squidfunk.github.io/mkdocs-material).

## Setup

```shell
cd docs
python3 -m venv .venv
source .venv/bin/activate
pip install mkdocs-material
```

## Local development

```shell
source .venv/bin/activate
mkdocs serve --dev-addr 0.0.0.0:8000 --livereload
```

The site will be available at [http://localhost:8000](http://localhost:8000).

## Build

```shell
source .venv/bin/activate
mkdocs build
```

The static site output will be in the `site/` directory.

## Deployment

The site is automatically deployed to GitHub Pages via GitHub Actions on push to `main`.
