# Open Data Manager Documentation

This repo contains documentation site for ODM product.

Documentation site is based on [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Markdown syntax

The documentation based on [Markdown Syntax](https://www.markdownguide.org/basic-syntax/). Also it supports [Extended Syntax](https://www.markdownguide.org/extended-syntax/).

Also there is a lot of additional markdown features. See the [reference](https://squidfunk.github.io/mkdocs-material/reference/).

If something does not work please feel free to contact with developers.

## Development

Install venv

```shell
rm -rf venv
python3 -m venv venv
source venv/bin/activate
```

Install dependencies

```sh
python3 -m pip install -r requirements.txt
```

To serve documentation on localhost:

```sh
mkdocs serve
```

## Linting

We use [pre-commit](https://pre-commit.com/) as a tool to maintain a unified documentation standard.

[This module](https://github.com/igorshubovych/markdownlint-cli) with [these rule sets](https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md) is used for linting documentation.

Install

```sh
brew install pre-commit
```

Run

```sh
pre-commit run -a
```

## Deployment

Deployment latest version is configured via Github Actions.

[Mike](https://squidfunk.github.io/mkdocs-material/setup/setting-up-versioning/#usage) plugin is used for deploying versions.

To deploy a new version create a new branch with pattern `release/*.*` (for example `release/1.54`) and push it.

A new folder will be created in `gh-pages` branch that will be available on Github Pages after some time.
