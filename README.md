# avtools-docs

Documentation site for **AV Tools** on CERN Kubernetes, published to
`avtools.docs.cern.ch` via GitLab Pages. Built with MkDocs Material (the CERN
default SSG), modelled on `kubernetes.docs.cern.ch`.

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8000
mkdocs build --strict   # what CI runs
```

## Publish to CERN GitLab Pages

1. Push this repo to a new GitLab project (e.g. `itdcim/avtools-docs`).
2. Register a site in **Web Services** → `avtools.docs.cern.ch`
   (see the CERN "Documentation How-to Guide" → *Create a site in Web Services*).
3. The `pages` job (via CERN's `mkdocs-ci` include) in `.gitlab-ci.yml` builds
   and publishes `public/` on the default branch; merge requests run
   `test:docs` (build-only) so broken docs fail the MR.

> **A green pipeline does not prove the site is live.** docs.cern.ch sites are
> served from a "safe storage" that GitLab only pushes to when `.gitlab-ci.yml`
> includes CERN's `mkdocs-ci` template. `mkdocs build --strict` passing (and a
> generic `pages` artifact) says nothing about whether that push happened, nor
> about whether the Web Services site is linked to this GitLab project
> ("Set up Pages" in the portal). Both must be true for content to appear.
> Likewise, changing `site_url` in `mkdocs.yml` only edits the built HTML — it
> never touches Web Services registration or linkage.

## Structure

```
mkdocs.yml            # theme + nav
requirements.txt      # mkdocs-material (pinned)
.gitlab-ci.yml        # CERN Pages build
docs/
  index.md
  getting-started.md
  architecture.md
  repos.md
  deployment/magnum.md
  secrets.md
  logging-monitoring.md
  operations.md
  faq.md
```

To reuse for **timeseries-DIP**: copy the tree into `timeseries-dip-docs`, swap the
content, register a new Web Services site.
