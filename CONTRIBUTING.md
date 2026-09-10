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

| Situation | Filename |
|-----------|----------|
| Plan linked to a Jira issue | `plans/<jira-key>.md` (lowercase, e.g. `plans/ocpstrat-3298.md`) |
| Plan without a Jira issue | `plans/<descriptive-slug>.md` (lowercase, hyphenated, e.g. `plans/csi-snapshot-validation.md`) |

### 2. Write the plan

Start from the [plan template](docs/format.md#plan-template) or copy an
existing plan from `plans/`. Every plan **must** include the YAML front-matter
metadata block — see [docs/format.md](docs/format.md#metadata-block) for
required and optional fields.

The body follows the IEEE 829-2008 structure. At minimum, include:

1. **Test Plan Identifier** — unique ID, version, date
2. **Introduction and Objectives** — purpose, acceptance criteria mapping
3. **Test Items and References** — PRs, code paths, Jira links
4. **Test Cases** — numbered test cases with steps and expected results

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

- **No secrets** — do not include credentials, tokens, API keys, kubeconfigs,
  or other sensitive material.
- **No customer data** — do not include customer names, cluster IDs, or
  support case references.
- **No private links** — all URLs must be publicly accessible or accessible
  to the intended audience (e.g. Red Hat Jira). Do not include links to
  private Slack channels, internal-only dashboards, or personal repositories.
- **Sanitize examples** — replace real cluster names, namespaces, and
  identifiers with placeholder values in any included command output.

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
