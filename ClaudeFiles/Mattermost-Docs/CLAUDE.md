# Mattermost Documentation

Sphinx-based documentation site published to https://docs.mattermost.com/.

## Build system

- **Sphinx** with Python 3.11+ and Pipenv
- `gmake html` — incremental build (fastest)
- `gmake clean html` — full rebuild
- `gmake livehtml` — live preview at http://127.0.0.1:8000
- `gmake linkcheck` — validate external links
- `gmake test` — run pytest for extension tests
- Output lands in `build/html/`; warnings in `build/warnings.log`
- Run `gmake clean` if builds become slow

## Directory structure

- `source/` — all documentation content
  - `index.rst` — root entry point with toctree
  - `conf.py` — Sphinx configuration (extensions, theme, redirects)
  - `redirects.py` — page redirect mappings
  - `administration-guide/` — server admin, config, compliance, deployment
  - `deployment-guide/` — installation and reference architecture
  - `end-user-guide/` — features, collaboration, preferences
  - `product-overview/` — capabilities, licensing
  - `integrations-guide/` — third-party integrations
  - `use-case-guide/` — mission-critical use cases
  - `security-guide/` — security and compliance
  - `agents/` — git submodule pointing to mattermost-plugin-agents
  - `_static/` — CSS, JS, fonts, images
  - `_templates/` — Jinja2 templates for Furo theme
  - `_generated/` — auto-generated files (created during build)
- `extensions/` — custom Sphinx extensions
- `scripts/` — utility scripts for doc processing
- `tests/` — pytest suite for custom extensions

## File formats

- **reStructuredText (.rst)** — primary format, ~94% of docs
- **Markdown (.md)** — supported via MyST Parser, ~6% of docs
- Both formats are valid; check neighboring files for which format a section uses

## Sphinx extensions

Built-in and third-party:
- `myst_parser` — Markdown support with `colon_fence` extension
- `sphinx.ext.autosectionlabel` — auto-generate section labels
- `sphinxcontrib.mermaid` — Mermaid.js diagrams
- `sphinx_copybutton` — copy buttons on code blocks
- `sphinx_inline_tabs` — tabbed content blocks

Custom (in `extensions/`):
- **reredirects** — fork of `sphinx_reredirects` with parallel read/write; manages page redirects
- **sitemap** — fork of `sphinx_sitemap` with parallel support; generates `sitemap.xml`
- **compass-icons** — downloads and renders Compass Icons
- **config-setting-v2** — structured documentation for configuration settings

## Theme

Furo — configured in `conf.py`. Custom CSS/JS lives in `source/_static/`.
Use `watch_and_sync.sh` for live CSS/JS reloading during theme development.

## Testing

- `make test` runs pytest against `tests/`
- Tests cover the custom `reredirects` extension (redirect handling, config validation, duplicate detection)
- Test fixtures in `tests/roots/`

## CI/CD

- **CI** (`.github/workflows/ci.yml`) — runs on push to `master` and all PRs: installs deps, runs tests, builds HTML, uploads artifacts
- **CD** (`.github/workflows/cd.yml`) — triggers after CI on `master`: syncs `build/html/` to `s3://docs.mattermost.com/`
- Submodules must be initialized — CI does `git submodule update --init --recursive`

## Key conventions

- Redirect old URLs via `source/redirects.py` when moving or renaming pages
- The `agents/` submodule is excluded from sitemap and redirect processing
- Section labels are auto-generated — avoid duplicate section titles within the same document
- Use tabs (`sphinx_inline_tabs`) for platform-specific or version-specific instructions
- Images go in `source/images/` or alongside their doc file
