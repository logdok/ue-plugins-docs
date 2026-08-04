# ue-plugins-docs

Public documentation site for Unreal Engine plugins by [logdok](https://github.com/logdok), built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and published via GitHub Pages.

**Live site:** https://logdok.github.io/ue-plugins-docs/

## How this repo works

The plugins themselves live in separate, private repositories. Each private repo's `Docs/` folder is synced into `docs/<plugin-name>/` here by a GitHub Actions workflow in that repo, on every push that touches its docs. Pushing to `main` here rebuilds and redeploys the site automatically.

**Don't edit `docs/<plugin-name>/` by hand** — those subfolders are overwritten by the next sync from the source repo. `docs/index.md`, `docs/<plugin-name>/index.md` (the per-plugin overview page) and `mkdocs.yml`'s navigation are hand-maintained here and are the right place to add a new plugin's top-level entry.

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve
```

## Adding a new plugin

1. Add a `docs/<plugin-name>/` entry to `mkdocs.yml`'s `nav:`.
2. Add a one-line entry to `docs/index.md`'s plugin list.
3. Set up a sync workflow in the plugin's own private repo (copy `push-docs.yml`'s pattern from the QuestSystem repo) with a fine-grained PAT scoped to this repo, stored as that repo's `DOCS_TOKEN` secret.
