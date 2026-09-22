# Agent instructions: quasarlab-docs

## Purpose

The public MkDocs Material site at `docs.herro.me`: homelab architecture, incident
postmortems, runbooks, ADRs and AI case studies. Everything here is world-readable.

## Boundaries

- **Pushing to `main` publishes to the public internet.** `.github/workflows/deploy.yml`
  runs `mkdocs build --strict` then `mkdocs gh-deploy --force --no-history` on every push
  to `main`, and it is live: runs succeeded on 2026-09-21.
- **The README is wrong about this.** It claims the workflow is "checked in but disabled
  (`if: false`)". There is no `if:` condition anywhere in the workflow. Trust the workflow
  file and the Actions tab, not the README, and fix the README when convenient.
- The deploy target is the `gh-pages` branch, with the custom domain set by
  `docs/CNAME`. Do not hand-edit `gh-pages`.
- A `concurrency: deploy-docs` group with `cancel-in-progress: false` serialises deploys
  so two quick pushes do not race on `gh-pages`.

## Validation

```
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs build --strict     # what CI gates on
mkdocs serve              # local preview
```

`--strict` turns warnings such as broken internal links into failures. Run it before
pushing, because the push is the publish.

## Landmines

- `Pygments==2.19.2` is pinned deliberately. 2.20.0 breaks `pymdownx.superfences` for
  plain ```` ```text ```` fences, which parse as inline code instead of a block and wreck
  the ASCII diagrams. Do not bump it casually.
- Content is public. Incident writeups routinely want real hostnames, IPs and topology;
  redact before committing rather than after.
- Adding a page without adding it to `mkdocs.yml` nav leaves it built but unreachable.

## Forbidden

- Do not publish credentials, tokens, real internal addresses, or unredacted screenshots.
- Do not push to `main` to "see if it builds". Run `mkdocs build --strict` locally.

## Completion

`mkdocs build --strict` passes locally, nav is updated if a page was added, and any real
identifiers are redacted. State explicitly that the change is going public.
