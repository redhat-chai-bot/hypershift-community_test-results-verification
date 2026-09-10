# HyperShift Test Results & Verification

Centralized repository for HyperShift test plans and pre-merge verification
reports. Replaces the previous pattern of self-hosting these artifacts in
individual personal GitHub repositories.

## Repository Structure

```
.
├── plans/                  # Test plans (IEEE 829-style Markdown)
│   ├── <jira-key>.md       # Plan linked to a Jira issue (e.g. ocpstrat-3298.md)
│   └── <slug>.md           # Plan without a Jira issue (e.g. csi-snapshot-validation.md)
├── reports/                # Pre-merge verification reports
│   └── <jira-key>/         # Reports grouped by Jira issue
│       └── <pr-number>.md  # One report per PR (e.g. 8698.md)
├── docs/                   # Repository documentation
│   ├── format.md           # Metadata model, format specification, relationships
│   └── migration.md        # Migration guide and inventory
├── CONTRIBUTING.md         # How to add plans and reports
└── README.md               # This file
```

### Plans (`plans/`)

Each test plan is a single Markdown file following the IEEE 829-2008 standard.
Plans are named by their **primary Jira issue key** in lowercase
(e.g. `ocpstrat-3298.md`). A single plan can reference multiple Jira issues
and PRs to support backport workflows — see [docs/format.md](docs/format.md)
for the metadata model.

Plans that do not have a Jira issue use a short descriptive slug instead
(e.g. `csi-snapshot-validation.md`).

### Reports (`reports/`)

Pre-merge verification reports capture the results of executing a test plan
against a specific PR. Reports are grouped into subdirectories named by Jira
issue key, with each report file named by the PR number it verifies.

### Documentation (`docs/`)

- **[docs/format.md](docs/format.md)** — Metadata model, naming conventions,
  format specification, and relationship model (1-to-N Jira/PR support).
- **[docs/migration.md](docs/migration.md)** — Guide for migrating existing
  test plans from personal repositories, including a migration inventory.

## Quick Start

1. Read the [contribution guide](CONTRIBUTING.md).
2. Review the [format specification](docs/format.md) for metadata requirements.
3. Copy an existing plan from `plans/` or use the template in
   [docs/format.md](docs/format.md#plan-template).
4. Open a PR adding your plan or report.

## Automation Integration

This repository is designed for integration with
[Chai Bot](https://github.com/openshift/ai-helpers) and the `ai-helpers`
skill library. The target workflow:

```
@chai-bot make a test plan for JIRA-1234 and add it to test-results-verification
```

See [docs/format.md](docs/format.md#automation-integration-contract) for the
integration contract that automation must follow when generating or updating
plans.

> **Status:** Automation integration is tracked in
> [CNTRLPLANE-4323](https://issues.redhat.com/browse/CNTRLPLANE-4323).
> The contract is documented; implementation in the `ai-helpers` skill
> library is a follow-up.

## Related Links

- [openshift/hypershift](https://github.com/openshift/hypershift) — main
  HyperShift repository
- [Jira: CNTRLPLANE-4323](https://issues.redhat.com/browse/CNTRLPLANE-4323) —
  tracking issue for this repository's structure
