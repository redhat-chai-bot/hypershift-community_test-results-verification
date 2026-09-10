# Migration Guide

This document provides guidance for migrating existing test plans from
personal repositories to this centralized repository.

## Overview

HyperShift team members have been self-hosting test plans and verification
reports in personal GitHub repositories. This centralized repository replaces
that pattern. All existing plans should be migrated here following the
conventions in [format.md](format.md) and [CONTRIBUTING.md](../CONTRIBUTING.md).

## Migration Steps

### 1. Inventory Your Plans

List all test plans and verification reports in your personal repository.
For each artifact, note:

- The Jira issue key (if any)
- The associated PRs
- Whether it is a plan, report, or both
- The current format (Markdown, HTML, etc.)

### 2. Convert to the Standard Format

For each plan:

1. **Add YAML front-matter** — see [format.md](format.md#metadata-block) for
   required fields.
2. **Rename the file** — use the naming convention
   (`<jira-key>.md` or `<slug>.md`).
3. **Restructure to IEEE 829** — if the plan does not already follow the IEEE
   829-2008 structure, reorganize it into the standard sections. At minimum,
   include: identifier, objectives, test items, and test cases.
4. **Sanitize content** — remove any secrets, customer data, private links,
   or real cluster identifiers. Replace with placeholders.

For HTML reports, convert to Markdown. If the HTML contains complex formatting
that does not convert cleanly, extract the key information (test results,
evidence links) into a Markdown report.

### 3. Open a Pull Request

- Branch from `main`.
- Add your converted files under `plans/` and/or `reports/`.
- Title: `Migrate test plan(s) from <source>`.
- In the PR description, note the source repository and any changes made
  during conversion.

### 4. Archive or Remove the Original

Once the PR is merged, archive or remove the plan from your personal
repository to avoid confusion. Consider adding a note or redirect pointing
to this repository.

## Migration Inventory

The table below tracks known existing test plans and their migration status.
Add entries as you discover plans in personal repos or other locations.

| Source | Jira Key | Description | Target File | Status |
|--------|----------|-------------|-------------|--------|
| [bryan-cox.github.io/architectural-artifact-sharing](https://bryan-cox.github.io/architectural-artifact-sharing/) | Various | Hosted verification reports | TBD (multiple files) | Not started |
| PR #1 in this repo | OCPSTRAT-3298 | NodePool rollout control test plan | `plans/ocpstrat-3298.md` | In progress (PR open) |
| PR #2 in this repo | OCPSTRAT-3150 | Etcd sharding test plan | `plans/ocpstrat-3150.md` | In progress (PR open) |

### Adding to the Inventory

When you discover existing test plans that should be migrated, add a row to
the table above via PR. Include:

- **Source** — link to the current location
- **Jira Key** — the associated Jira issue, if any
- **Description** — brief description of what the plan covers
- **Target File** — the expected filename in this repo
- **Status** — one of: `Not started`, `In progress`, `Migrated`, `Skipped`

## Tips

- **Don't migrate everything at once.** Start with active plans that are
  referenced by in-flight PRs.
- **Preserve authorship.** Use `git commit --author` if you are migrating
  someone else's plan and want to preserve their attribution.
- **Link back.** After migration, update any Jira issues that reference the
  old location to point to the new file in this repository.
- **Ask for help.** If a plan is in a format you are unsure how to convert,
  open an issue and the team can assist — or use Chai Bot's test plan
  generation skill to create a fresh plan from the Jira issue.
