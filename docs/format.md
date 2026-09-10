# Format Specification

This document defines the metadata model, naming conventions, file format, and
relationship model for test plans and verification reports in this repository.

## Naming Conventions

### Plans

| Situation | Pattern | Example |
|-----------|---------|---------|
| Linked to a Jira issue | `plans/<jira-key>.md` | `plans/ocpstrat-3298.md` |
| No Jira issue | `plans/<descriptive-slug>.md` | `plans/csi-snapshot-validation.md` |

Rules:

- Filenames are **lowercase**.
- Jira keys use the original project prefix and number (e.g. `ocpstrat-3298`,
  `cntrlplane-1234`, `ocpbugs-5678`).
- Slugs for plans without a Jira issue use lowercase words separated by
  hyphens. Keep them short and descriptive.
- One plan per file. Do not combine unrelated features.

### Reports

```
reports/<jira-key>/<pr-number>.md
```

Example: `reports/ocpstrat-3298/8698.md` verifies PR #8698 against the
test plan for OCPSTRAT-3298.

## Metadata Block

Every plan file **must** begin with a YAML front-matter block. This block
enables automation to discover, index, and link plans.

### Required Fields

```yaml
---
id: TP-OCPSTRAT-3298-001        # Unique plan identifier
version: "1.0"                   # Plan version (semver-style)
date: 2026-09-07                 # Last-updated date (ISO 8601)
title: "Predictable NodePool Rollout Control"
jira_issues:                     # Primary + backport Jira issues
  - key: OCPSTRAT-3298
    role: primary                # "primary" or "backport"
    url: https://issues.redhat.com/browse/OCPSTRAT-3298
pull_requests:                   # Associated PRs
  - repo: openshift/hypershift
    number: 8698
    url: https://github.com/openshift/hypershift/pull/8698
    role: implementation         # "implementation", "backport", or "test"
status: draft                    # "draft", "active", "superseded", or "archived"
---
```

### Optional Fields

```yaml
---
superseded_by: ocpstrat-3298-v2  # Filename (without .md) of the replacement plan
tags:                            # Freeform tags for discovery
  - nodepool
  - rollout
  - machine-config
ieee_829: true                   # Indicates IEEE 829-2008 structure
author: ""                       # Plan author (name or GitHub handle)
---
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique identifier. Convention: `TP-<JIRA-KEY>-<seq>` or `TP-<SLUG>-<seq>`. |
| `version` | string | yes | Plan revision. Increment on substantive changes. |
| `date` | date | yes | Last-updated date in ISO 8601 format. |
| `title` | string | yes | Human-readable plan title. |
| `jira_issues` | list | yes* | List of associated Jira issues. *May be empty for plans without a Jira issue. |
| `jira_issues[].key` | string | yes | Jira issue key (e.g. `OCPSTRAT-3298`). |
| `jira_issues[].role` | string | yes | `primary` or `backport`. Exactly one issue should be `primary`. |
| `jira_issues[].url` | string | yes | Full URL to the Jira issue. |
| `pull_requests` | list | yes* | List of associated PRs. *May be empty if PRs are not yet opened. |
| `pull_requests[].repo` | string | yes | GitHub repository in `org/repo` format. |
| `pull_requests[].number` | integer | yes | PR number. |
| `pull_requests[].url` | string | yes | Full URL to the PR. |
| `pull_requests[].role` | string | yes | `implementation`, `backport`, or `test`. |
| `status` | string | yes | Plan lifecycle status (see below). |
| `superseded_by` | string | no | Filename stem of the replacement plan. |
| `tags` | list | no | Freeform tags for search and filtering. |
| `ieee_829` | boolean | no | `true` if the plan follows IEEE 829-2008 structure. |
| `author` | string | no | Plan author identifier. |

### Status Lifecycle

```
draft → active → superseded
                → archived
```

- **draft** — plan is under review or incomplete.
- **active** — plan is approved and in use for verification.
- **superseded** — replaced by a newer plan (set `superseded_by`).
- **archived** — no longer relevant (feature removed, etc.).

## One-to-Many Relationships

### One Plan, Many Jira Issues (Backports)

A feature implemented in a main-branch PR often gets backported to one or
more release branches. Each backport has its own Jira issue and PR, but the
test plan is the same. Rather than duplicating the plan file, list all
related Jira issues and PRs in the metadata:

```yaml
jira_issues:
  - key: OCPSTRAT-3298
    role: primary
    url: https://issues.redhat.com/browse/OCPSTRAT-3298
  - key: OCPBUGS-45678
    role: backport
    url: https://issues.redhat.com/browse/OCPBUGS-45678
  - key: OCPBUGS-45679
    role: backport
    url: https://issues.redhat.com/browse/OCPBUGS-45679

pull_requests:
  - repo: openshift/hypershift
    number: 8698
    url: https://github.com/openshift/hypershift/pull/8698
    role: implementation
  - repo: openshift/hypershift
    number: 8750
    url: https://github.com/openshift/hypershift/pull/8750
    role: backport
```

The plan file is named after the **primary** Jira issue key.

### One Plan, Many PRs (Same Feature)

Some features span multiple PRs (e.g. API change + controller + tests).
List all PRs in the `pull_requests` field.

### Plans Without a Jira Issue

If there is no Jira issue, leave `jira_issues` as an empty list and name
the file with a descriptive slug:

```yaml
jira_issues: []
pull_requests:
  - repo: openshift/hypershift
    number: 9100
    url: https://github.com/openshift/hypershift/pull/9100
    role: implementation
```

## Plan Body Structure

Plans should follow the IEEE 829-2008 standard structure. The recommended
sections are:

1. **Test Plan Identifier** — ID, version, date, IEEE 829 reference
2. **Introduction and Objectives** — purpose, objectives, acceptance criteria
   mapping, automation-first principle
3. **Test Items and References** — implementation PRs, code paths, API
   changes, related documentation
4. **Test Environment** — cluster types, platforms, prerequisites
5. **Test Cases** — numbered cases with preconditions, steps, expected results,
   and automation references
6. **Risks and Mitigations** — known risks and contingency plans
7. **Approvals** — sign-off section

This structure is a guideline. Adapt sections as needed for the feature under
test — not every plan needs every section.

## Plan Template

Use this template to start a new test plan:

````markdown
---
id: TP-<JIRA-KEY>-001
version: "1.0"
date: YYYY-MM-DD
title: "<Feature Name>"
jira_issues:
  - key: <JIRA-KEY>
    role: primary
    url: https://issues.redhat.com/browse/<JIRA-KEY>
pull_requests:
  - repo: openshift/hypershift
    number: <PR-NUMBER>
    url: https://github.com/openshift/hypershift/pull/<PR-NUMBER>
    role: implementation
status: draft
tags: []
ieee_829: true
---

# Test Plan: <Feature Name>

## 1. Test Plan Identifier

**ID:** TP-<JIRA-KEY>-001
**Version:** 1.0
**Date:** YYYY-MM-DD
**IEEE 829 Compliance:** This document follows the IEEE 829-2008 Standard
for Software and System Test Documentation.

---

## 2. Introduction and Objectives

### 2.1 Purpose

Describe the feature under test and the purpose of this plan.

### 2.2 Objectives

1. Objective 1
2. Objective 2

### 2.3 Acceptance Criteria Mapping

| Acceptance Criterion | Test Case |
|----------------------|-----------|
| AC #1 — description | TC-001 (Section 5.1) |

---

## 3. Test Items and References

| Item | Reference |
|------|-----------|
| Implementation PR | [openshift/hypershift#NNNN](https://github.com/openshift/hypershift/pull/NNNN) |
| Feature Jira | [JIRA-KEY](https://issues.redhat.com/browse/JIRA-KEY) |

---

## 4. Test Environment

| Property | Value |
|----------|-------|
| Cluster type | HostedCluster on AWS |
| OCP version | 4.x |
| Management cluster | ... |

---

## 5. Test Cases

### 5.1 TC-001: <Test Case Title>

**Objective:** What this test verifies.

**Preconditions:**
- Precondition 1

**Primary method (automated):**
- Test file: `test/e2e/...`
- Run with: `make e2e TEST_NAME=...`

**Fallback method (manual):**

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | ... | ... |

---

## 6. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| ... | ... |

---

## 7. Approvals

| Role | Name | Date |
|------|------|------|
| Author | | |
| Reviewer | | |
````

## Report Format

Verification reports are lighter-weight than plans. A report captures the
results of executing a plan against a specific PR.

### Report Template

````markdown
---
plan: ocpstrat-3298              # Filename stem of the test plan
jira_key: OCPSTRAT-3298
pr:
  repo: openshift/hypershift
  number: 8698
  url: https://github.com/openshift/hypershift/pull/8698
date: YYYY-MM-DD
tester: ""
verdict: pass                    # "pass", "fail", or "conditional"
---

# Verification Report: PR #8698

## Environment

| Property | Value |
|----------|-------|
| Cluster type | HostedCluster on AWS |
| OCP version | 4.18 |
| Platform | AWS us-east-1 |

## Results

| Test Case | Result | Notes |
|-----------|--------|-------|
| TC-001 | Pass | Ran via CI: [link] |
| TC-002 | Pass | |

## Evidence

- CI job: [link to Prow job]
- Logs: [link or summary]

## Verdict

**Pass** — all test cases passed.
````

## Automation Integration Contract

This section defines the contract that automation tools (e.g. Chai Bot,
ai-helpers skills) must follow when creating or updating plans and reports
in this repository.

### Creating a Plan

Automation **must**:

1. Generate a valid Markdown file with the required YAML front-matter
   metadata block.
2. Use the correct filename convention (`plans/<jira-key>.md` or
   `plans/<slug>.md`).
3. Follow the IEEE 829-2008 body structure.
4. Set `status: draft` for newly generated plans.
5. Include all known Jira issues and PRs in the metadata.
6. Open a PR against `main` with title format
   `<JIRA-KEY>: Add test plan for <feature>`.

Automation **must not**:

- Commit directly to `main` — always use a PR.
- Include secrets, credentials, customer data, or private links.
- Overwrite an existing plan without incrementing the `version` field and
  updating the `date`.
- Create duplicate plan files for backport Jira issues — instead, update
  the existing plan's `jira_issues` list.

### Updating a Plan

When adding a backport Jira issue or PR to an existing plan, automation
should:

1. Read the existing plan file.
2. Append the new Jira issue or PR to the appropriate list.
3. Increment the `version` field.
4. Update the `date` field.
5. Open a PR with the changes.

### Creating a Report

Automation should follow the report format defined above and place the
file at `reports/<jira-key>/<pr-number>.md`.

### File Validation

Automation should validate before committing:

- YAML front-matter is parseable.
- Required metadata fields are present.
- All URLs are well-formed.
- Filename matches the primary Jira key or slug.
- No secrets or private content patterns detected.

> **Note:** Implementation of these automation capabilities in the
> `ai-helpers` skill library is tracked as a follow-up to
> [CNTRLPLANE-4323](https://issues.redhat.com/browse/CNTRLPLANE-4323).
> The contract above is stable and ready for implementation.
