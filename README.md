# Quasarlab Docs

Architecture, runbooks, incidents, and command references for my homelab.

Site: <https://docs.herro.me>

## What this is

A working portfolio of how I run a small home K8s + Proxmox lab. The goal is to show how the pieces fit together and how I think about failures, not to be an exhaustive command index. The four sections in priority order:

1. **Architecture** — what is where and why.
2. **Incidents** — real failures, walked end to end.
3. **Runbooks** — short fix-it docs derived from real incidents.
4. **Decisions (ADRs)** — short notes on tradeoffs taken.
5. **Cheatsheet** — commands and one-liners I actually use, organized by tool.

## Running locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

## Deploy

Build locally and push the rendered site to the `gh-pages` branch, which is the configured Pages source:

```bash
. .venv/bin/activate
mkdocs gh-deploy --force --no-history
```

The custom domain `docs.herro.me` is set via the `docs/CNAME` file (committed in the source) and propagates into the built site automatically.

There is a `.github/workflows/deploy.yml` workflow checked in but **disabled** (`if: false`). It produced broken fenced-code rendering in CI for unknown reasons (identical Python and pip versions to the local venv, but `pymdownx.superfences` did not load). Re-enable it once the root cause is found.
