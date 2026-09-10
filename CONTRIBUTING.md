# Contributing

This guide covers how to add test plans and verification reports to this
repository.

## Before You Start

- Read the [format specification](docs/format.md) for metadata requirements
  and the relationship model.
- Check `plans/` for an existing plan that covers your feature — you may only
  need to add a report, not a new plan.

## Adding a Test Plan

### 1. Choose a filename

Use the naming conventions in [docs/format.md](docs/format.md#naming-conventions):
lowercase Jira key (e.g. `plans/ocpstrat-3298.md`) or descriptive slug
(e.g. `plans/csi-snapshot-validation.md`).

### 2. Write the plan

Start from the [plan template](docs/format.md#plan-template) or copy an
existing plan from `plans/`. Every plan **must** include the YAML front-matter
metadata block — see [docs/format.md](docs/format.md#metadata-block) for
required and optional fields.

The body follows the IEEE 829-2008 structure. See
[docs/format.md](docs/format.md#plan-body-structure) for the recommended
sections and template.

### 3. Open a pull request

- Branch from `main`.
- Add your file under `plans/`.
- Use a descriptive PR title: `<JIRA-KEY>: Add test plan for <feature>`.
- Request review from at least one team member.

## Adding a Verification Report

### 1. Create the report directory (if needed)

```
reports/<jira-key>/
```

### 2. Write the report

Name the file by the PR number it verifies:

```
reports/<jira-key>/<pr-number>.md
```

Include:

- **Header** — PR link, plan reference, date, tester
- **Environment** — cluster type, OCP version, platform
- **Results** — pass/fail per test case from the plan
- **Evidence** — links to CI runs, logs, screenshots
- **Verdict** — overall pass/fail and any caveats

### 3. Open a pull request

Use title format: `<JIRA-KEY>: Add verification report for PR #<number>`.

## Backports and One-to-Many Relationships

A single test plan often applies to multiple Jira issues (e.g. a feature
issue plus its backport issues) and multiple PRs. The metadata block in
each plan supports this via the `jira_issues` and `pull_requests` lists.
See [docs/format.md](docs/format.md#one-to-many-relationships) for details.

Do **not** duplicate a plan file for each backport. Instead, add the
additional Jira issue keys and PR links to the existing plan's metadata.

## Content Rules

Do not include secrets, customer data, or private links. Sanitize all
examples by replacing real identifiers with placeholders. See the
[automation integration contract](docs/format.md#automation-integration-contract)
for the full list of content prohibitions.

## Style Guidelines

- Use Markdown with standard GitHub-Flavored Markdown (GFM) extensions.
- Wrap lines at a reasonable length for readability in diffs (no strict limit).
- Use ATX-style headers (`#`, `##`, etc.).
- Use fenced code blocks with language identifiers.
- Prefer tables for structured data (acceptance criteria mappings, test
  matrices).

## Questions

If anything is unclear, open an issue on this repository or ask in the
HyperShift team channel.
