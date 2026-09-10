# Agent Guide

This repository stores HyperShift **test plans** and **verification reports**.
For full details see [CONTRIBUTING.md](CONTRIBUTING.md),
[docs/format.md](docs/format.md), and [docs/migration.md](docs/migration.md).

## Non-Negotiable Rules

1. **Plans (`plans/`)** — IEEE 829-style Markdown only, one file per feature.
   Each file must have YAML front-matter with required fields (`id`, `version`,
   `date`, `title`, `jira_issues`, `pull_requests`, `status`).

2. **Reports (`reports/`)** — HTML bundles are the preferred and authoritative
   format. A bundle directory must contain `metadata.yaml` and `index.html`;
   it may also include `scenario-N.html` pages, `appendices.html`,
   `automated-tests.html`, an `assets/` folder, and a `scripts/` folder.
   Markdown reports (`<pr-number>.md`) are also supported.

3. **Derived Markdown is optional** — `derived/report.md` is a convenience
   copy; the HTML bundle is always authoritative. Never delete HTML files
   after generating a Markdown summary.

4. **Content sanitization** — never commit secrets, credentials, bearer
   tokens, private keys, PII, customer data, or private/internal URLs.
   Replace real identifiers with placeholders.

5. **Preserve existing scope** — do not convert HTML bundles to Markdown
   during migration. Validate the complete diff before pushing.
