# Agent Guide

This repository stores HyperShift **test plans** and **verification reports**.
For full details see [CONTRIBUTING.md](CONTRIBUTING.md),
[docs/format.md](docs/format.md), and [docs/migration.md](docs/migration.md).

## Non-Negotiable Rules

1. **Plans (`plans/`)** — Markdown with YAML front-matter, one file per feature.
   Two profiles are supported:
   - **IEEE 829** (`ieee_829: true` in front-matter) — full IEEE 829-2008
     body structure with numbered sections. Preferred for comprehensive plans.
   - **Lightweight** (default when `ieee_829` is absent or `false`) — free-form
     Markdown covering scope, test cases, and acceptance criteria.
   Required metadata (discoverability core): `id`, `title`, `date`, `status`.
   Recommended when applicable: `version`, `jira_issues`, `pull_requests`.
   Legacy/incomplete artifacts may omit unavailable metadata — document gaps,
   do not fabricate facts.

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
