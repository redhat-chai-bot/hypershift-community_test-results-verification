# Test Plan: Predictable NodePool Rollout Control

## 1. Test Plan Identifier

**ID:** TP-OCPSTRAT-9238-001
**Version:** 1.0
**Date:** 2026-09-05
**IEEE 829 Compliance:** This document follows the IEEE 829-2008 Standard for Software and System Test Documentation.

---

## 2. Introduction and Objectives

### 2.1 Purpose

This test plan defines the manual verification strategy for the **Predictable NodePool Rollout Control** feature in HyperShift. The feature decouples NodePool rollout triggering from management-side configuration changes (e.g., HAProxy image digest bumps) so that only user-driven spec changes cause worker node replacement.

### 2.2 Objectives

1. Verify that management-side-only configuration changes (e.g., HAProxy image annotation updates) do **not** trigger a NodePool rollout or node replacement.
2. Verify that spec-driven configuration changes (e.g., adding a user MachineConfig) **do** trigger a NodePool rollout and update the rollout hash annotation.
3. Verify that after an operator upgrade (simulated by removing the rollout config annotation), the controller re-seeds the annotation on the next reconciliation **without** triggering a rollout.

### 2.3 Jira Issue Availability

> **Note:** Jira issue [OCPSTRAT-9238](https://issues.redhat.com/browse/OCPSTRAT-9238) could not be retrieved at the time of this plan's creation (the Jira API returned an access error, indicating the issue may not exist, may have a restricted security level, or the service account lacks project access). Acceptance-criteria mapping from OCPSTRAT-9238 is therefore **unavailable** and has not been fabricated. The implementation PR itself references [OCPSTRAT-3298](https://issues.redhat.com/browse/OCPSTRAT-3298) ("Predictable NodePool rollout control") as the underlying feature issue. All test scenarios in this plan are derived exclusively from the PR description, code diff, and the e2e test file.

---

## 3. Test Items and References

| Item | Reference |
|------|-----------|
| Implementation PR | [openshift/hypershift#8698](https://github.com/openshift/hypershift/pull/8698) — "CNTRLPLANE-3632: Predictable NodePool rollout control" |
| E2E test file | `test/e2e/v2/tests/nodepool_rollout_control_test.go` (new, 397 lines) |
| API changes | `api/hypershift/v1beta1/nodepool_conditions.go` — new `ConfigUpdatePending` condition type, `ManagementConfigDriftReason` reason |
| Controller: config hashing | `hypershift-operator/controllers/nodepool/config.go` — new `RolloutHash()` / `RolloutHashWithoutVersion()` methods |
| Controller: CAPI propagation | `hypershift-operator/controllers/nodepool/capi.go` — refactored `propagateVersionAndTemplate()` |
| Controller: conditions | `hypershift-operator/controllers/nodepool/conditions.go` — new `configUpdatePendingCondition()`, updated `updatingConfigCondition()` |
| Controller: token lifecycle | `hypershift-operator/controllers/nodepool/token.go` — rewritten `isOutdated()` |
| Controller: main reconciler | `hypershift-operator/controllers/nodepool/nodepool_controller.go` — annotation seeding on first reconcile |
| Karpenter integration | `karpenter-operator/controllers/karpenterignition/karpenterignition_controller.go` — parallel rollout config annotation tracking |
| Unit tests | `capi_test.go` (+670), `conditions_test.go` (+138), `config_test.go` (+516), `nodepool_controller_test.go` (+12) |
| Feature Jira | [OCPSTRAT-3298](https://issues.redhat.com/browse/OCPSTRAT-3298) (referenced by the PR) |
| Test plan Jira | [OCPSTRAT-9238](https://issues.redhat.com/browse/OCPSTRAT-9238) (inaccessible; see Section 2.3) |

### 3.1 Key Annotations

| Annotation | Purpose |
|------------|---------|
| `hypershift.openshift.io/nodePoolCurrentRolloutConfig` | Tracks the rollout-relevant config hash (spec-driven inputs only). Changed only on spec-driven rollouts. |
| `hypershift.openshift.io/nodePoolCurrentConfig` | Tracks the full config hash (all inputs including management-side). Changes on any input change. |
| `hypershift.openshift.io/nodePoolCurrentConfigVersion` | Legacy full config+version hash. Retained for backward compatibility. |

### 3.2 Key Conditions

| Condition Type | Meaning |
|----------------|---------|
| `UpdatingConfig` | `True` when the rollout hash differs from the stored annotation (a spec-driven rollout is in progress). `False` when stable. |
| `ConfigUpdatePending` | `True` when management-side content has drifted but no rollout has been triggered. `False` otherwise. |

---

## 4. Features to Be Tested

1. **Management-side image isolation** — Changing the HAProxy image annotation on a NodePool must not alter the `nodePoolCurrentRolloutConfig` annotation, must not set `UpdatingConfig` to `True`, and must not replace any nodes.
2. **Spec-driven rollout triggering** — Adding a user MachineConfig to a NodePool's `spec.config` must trigger a full rollout, update both `nodePoolCurrentRolloutConfig` and `nodePoolCurrentConfig` annotations to new values, and result in nodes running the new configuration.
3. **Operator upgrade annotation seeding** — When the `nodePoolCurrentRolloutConfig` annotation is absent (simulating a pre-upgrade NodePool), the controller must re-seed it on the next reconciliation without triggering a rollout or replacing nodes.

---

## 5. Features Not to Be Tested

The following are explicitly out of scope for this manual test plan:

- **Version upgrade rollouts** — Release image upgrades (version field changes) are an existing feature and are not modified by this PR.
- **Karpenter-specific rollout behavior** — The Karpenter integration changes (`karpenterignition_controller.go`) parallel the CAPI changes but target `OpenshiftEC2NodeClass` objects. Karpenter-specific e2e coverage is not included in the PR's test file and is out of scope here.
- **InPlace upgrade strategy** — The e2e tests exercise `Replace` (RollingUpdate) strategy only. InPlace rollout behavior is not covered.
- **KubeVirt platform** — The spec-driven rollout test explicitly skips KubeVirt (pending [CNV-38196](https://issues.redhat.com/browse/CNV-38196)).
- **ConfigUpdatePending condition validation** — The new `ConfigUpdatePending` condition is unit-tested but not exercised in the e2e tests; manual verification of this condition is out of scope.
- **Multi-replica rollout dynamics** — Tests use 1-replica NodePools; surge/unavailability behavior at scale is not covered.
- **Build, deployment, or CI infrastructure** — This plan covers test execution only.

---

## 6. Test Approach and Design

### 6.1 Approach

Each test scenario is executed manually against a running HyperShift management cluster with at least one HostedCluster and a default NodePool. The tester uses `oc` or `kubectl` commands against the management cluster and verifies outcomes through annotation inspection, condition checks, and node identity comparisons. The scenarios mirror the three automated e2e tests in `nodepool_rollout_control_test.go`.

### 6.2 Common Prerequisites

- A running HyperShift management cluster with an operational HostedCluster.
- The HyperShift operator version includes the changes from PR #8698 (or equivalent).
- A default NodePool exists with at least one ready node.
- The default NodePool has the `hypershift.openshift.io/nodePoolCurrentRolloutConfig` annotation set by the controller.
- The `UpdatingConfig` condition on the default NodePool is `False`.
- `oc` / `kubectl` CLI access to both the management cluster and the hosted cluster.
- Sufficient permissions to create/patch/delete NodePool objects, ConfigMaps, and read Node objects in the hosted cluster.

---

### 6.3 Test Case TC-001: Management-Side Image Change Must Not Trigger Rollout

**Objective:** Confirm that patching the HAProxy image annotation on a NodePool does not trigger a rollout, does not change the rollout hash, and does not replace any nodes.

**Prerequisites:**
- Common prerequisites (Section 6.2) are satisfied.
- Identify the default NodePool name: `oc get nodepools -n <hc-namespace>`.

**Steps:**

1. **Record baseline state:**
   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Record this value as `BASELINE_ROLLOUT_HASH`.

   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentConfig}'
   ```
   Record this value as `BASELINE_CONFIG_HASH`.

   Verify `UpdatingConfig` is `False`:
   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.status.conditions[?(@.type=="UpdatingConfig")].status}'
   ```
   Expected: `False`.

   Record baseline node names (on the hosted cluster):
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=<np-name> -o jsonpath='{.items[*].metadata.name}'
   ```
   Record as `BASELINE_NODES`.

2. **Patch the HAProxy image annotation:**
   ```
   oc annotate nodepool <np-name> -n <hc-namespace> \
     hypershift.openshift.io/haproxyImage=quay.io/openshift/origin-haproxy-router:e2e-dummy-digest \
     --overwrite
   ```

3. **Observe stability over 2 minutes (check every 15 seconds):**

   Repeat the following checks at 15-second intervals for at least 2 minutes:

   a. Rollout config annotation unchanged:
   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Expected: equals `BASELINE_ROLLOUT_HASH`.

   b. `UpdatingConfig` condition remains `False`:
   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.status.conditions[?(@.type=="UpdatingConfig")].status}'
   ```
   Expected: `False`.

   c. Node count unchanged (on the hosted cluster):
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=<np-name> --no-headers | wc -l
   ```
   Expected: same count as baseline.

4. **Verify node identity is preserved:**
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=<np-name> -o jsonpath='{.items[*].metadata.name}'
   ```
   Expected: identical set to `BASELINE_NODES` (no replacements occurred).

5. **Verify final rollout hash:**
   ```
   oc get nodepool <np-name> -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Expected: equals `BASELINE_ROLLOUT_HASH`.

6. **Clean up:**
   ```
   oc annotate nodepool <np-name> -n <hc-namespace> hypershift.openshift.io/haproxyImage-
   ```

**Pass criteria:** All assertions in steps 3-5 hold. No rollout triggered; no nodes replaced; rollout hash stable.

**Fail criteria:** Any of: `UpdatingConfig` becomes `True`; rollout hash changes; node names change; node count changes.

---

### 6.4 Test Case TC-002: Spec-Driven MachineConfig Change Must Trigger Rollout

**Objective:** Confirm that adding a MachineConfig to a NodePool's `spec.config` triggers a rollout, updates both hash annotations to new values, and results in nodes converging on the new configuration.

**Prerequisites:**
- Common prerequisites (Section 6.2) are satisfied.
- The platform is **not** KubeVirt (this test is skipped for KubeVirt per CNV-38196).

**Steps:**

1. **Create a dedicated test NodePool:**

   Create a NodePool manifest with 1 replica and RollingUpdate strategy (MaxUnavailable=0, MaxSurge=1), based on the default NodePool's platform configuration. Example:
   ```yaml
   apiVersion: hypershift.openshift.io/v1beta1
   kind: NodePool
   metadata:
     name: rollout-ctl-test
     namespace: <hc-namespace>
   spec:
     clusterName: <hosted-cluster-name>
     replicas: 1
     release:
       image: <same-as-default-nodepool>
     platform: <copy-from-default-nodepool>
     management:
       upgradeType: Replace
       replace:
         strategy: RollingUpdate
         rollingUpdate:
           maxUnavailable: 0
           maxSurge: 1
   ```
   Apply: `oc apply -f <nodepool-manifest>.yaml`

2. **Wait for the NodePool to become ready:**
   ```
   oc wait nodepool rollout-ctl-test -n <hc-namespace> --for=condition=Ready --timeout=20m
   ```
   Verify at least one ready node exists (on the hosted cluster):
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=rollout-ctl-test --no-headers | wc -l
   ```
   Expected: `1`.

3. **Record baseline state:**
   ```
   oc get nodepool rollout-ctl-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Record as `BASELINE_ROLLOUT_HASH`.

   ```
   oc get nodepool rollout-ctl-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentConfig}'
   ```
   Record as `BASELINE_CONFIG_HASH`.

4. **Create a MachineConfig ConfigMap:**
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: rollout-ctl-mc
     namespace: <hc-namespace>
   data:
     config: |
       apiVersion: machineconfiguration.openshift.io/v1
       kind: MachineConfig
       metadata:
         name: predictable-rollout-test
         labels:
           machineconfiguration.openshift.io/role: worker
       spec:
         config:
           ignition:
             version: "3.2.0"
           storage:
             files:
               - path: /etc/predictable-rollout-test
                 contents:
                   source: "data:,rollout-test%0A"
   ```
   Apply: `oc apply -f <configmap-manifest>.yaml`

5. **Patch the NodePool to reference the MachineConfig:**
   ```
   oc patch nodepool rollout-ctl-test -n <hc-namespace> --type=merge \
     -p '{"spec":{"config":[{"name":"rollout-ctl-mc"}]}}'
   ```

6. **Wait for the rollout to complete:**

   Monitor `UpdatingConfig` condition until it returns to `False` (may take 10-20 minutes depending on platform):
   ```
   oc wait nodepool rollout-ctl-test -n <hc-namespace> --for=condition=UpdatingConfig=False --timeout=30m
   ```
   Then wait for all nodes to be ready:
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=rollout-ctl-test -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}'
   ```
   Expected: `True` for all nodes.

7. **Verify the rollout hash annotation was updated:**
   ```
   oc get nodepool rollout-ctl-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Expected: **differs** from `BASELINE_ROLLOUT_HASH`; is non-empty.

8. **Verify the full config hash annotation was also updated:**
   ```
   oc get nodepool rollout-ctl-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentConfig}'
   ```
   Expected: **differs** from `BASELINE_CONFIG_HASH`; is non-empty.

9. **Clean up:**
   ```
   oc delete nodepool rollout-ctl-test -n <hc-namespace>
   oc delete configmap rollout-ctl-mc -n <hc-namespace>
   ```

**Pass criteria:** Rollout completes successfully; both `nodePoolCurrentRolloutConfig` and `nodePoolCurrentConfig` annotations have changed to new non-empty values.

**Fail criteria:** Rollout does not trigger; annotation values remain at baseline; nodes never converge to ready state.

---

### 6.5 Test Case TC-003: Operator Upgrade Annotation Seeding Without Rollout

**Objective:** Confirm that when the `nodePoolCurrentRolloutConfig` annotation is absent (simulating a pre-upgrade NodePool), the controller re-seeds it on the next reconciliation without triggering a rollout or replacing nodes.

**Prerequisites:**
- Common prerequisites (Section 6.2) are satisfied.

**Steps:**

1. **Create a dedicated test NodePool:**

   Create a NodePool with 1 replica (using defaults for upgrade strategy). Apply and wait for ready:
   ```
   oc apply -f <nodepool-manifest>.yaml
   oc wait nodepool upgrade-seed-test -n <hc-namespace> --for=condition=Ready --timeout=20m
   ```

2. **Record baseline state:**

   Verify the rollout config annotation exists:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Expected: non-empty. Record as `BASELINE_ROLLOUT_HASH`.

   Record `nodePoolCurrentConfig`:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentConfig}'
   ```
   Record as `BASELINE_CONFIG_HASH`.

   Verify `UpdatingConfig` is `False`:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.status.conditions[?(@.type=="UpdatingConfig")].status}'
   ```
   Expected: `False`.

   Record baseline node names (on the hosted cluster):
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=upgrade-seed-test -o jsonpath='{.items[*].metadata.name}'
   ```
   Record as `BASELINE_NODES`.

3. **Simulate pre-upgrade state by removing the rollout config annotation:**
   ```
   oc annotate nodepool upgrade-seed-test -n <hc-namespace> \
     hypershift.openshift.io/nodePoolCurrentRolloutConfig-
   ```

4. **Force a reconciliation via a dummy annotation:**
   ```
   oc annotate nodepool upgrade-seed-test -n <hc-namespace> \
     hypershift.openshift.io/e2e-reconcile-trigger="$(date +%s)" --overwrite
   ```

5. **Wait for the controller to re-seed the rollout config annotation (up to 1 minute):**

   Poll every 10 seconds:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentRolloutConfig}'
   ```
   Expected: a non-empty value appears within 1 minute.

6. **Verify no rollout was triggered (observe over 2 minutes, every 15 seconds):**

   a. `UpdatingConfig` condition remains `False`:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.status.conditions[?(@.type=="UpdatingConfig")].status}'
   ```
   Expected: `False`.

   b. `nodePoolCurrentConfig` annotation is unchanged:
   ```
   oc get nodepool upgrade-seed-test -n <hc-namespace> -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/nodePoolCurrentConfig}'
   ```
   Expected: equals `BASELINE_CONFIG_HASH`.

   c. Node count unchanged (on the hosted cluster):
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=upgrade-seed-test --no-headers | wc -l
   ```
   Expected: same count as baseline.

7. **Verify node identity is preserved:**
   ```
   oc get nodes -l hypershift.openshift.io/nodePool=upgrade-seed-test -o jsonpath='{.items[*].metadata.name}'
   ```
   Expected: identical set to `BASELINE_NODES` (no replacements occurred).

8. **Clean up:**
   ```
   oc delete nodepool upgrade-seed-test -n <hc-namespace>
   ```

**Pass criteria:** Rollout config annotation is re-seeded within 1 minute; no rollout triggered; `UpdatingConfig` stays `False`; node identities unchanged; full config hash unchanged.

**Fail criteria:** Annotation is not re-seeded within 1 minute; `UpdatingConfig` becomes `True`; nodes are replaced; config hash changes.

---

## 7. Pass/Fail Criteria

### 7.1 Overall Pass

All three test cases (TC-001, TC-002, TC-003) pass their individual criteria as defined in Sections 6.3-6.5.

### 7.2 Overall Fail

Any single test case fails its criteria.

### 7.3 Evidence Requirements

For each test case, the tester must capture:
- Screenshots or terminal output of all `oc get` commands showing annotation values and condition statuses.
- Timestamped logs of the 2-minute observation windows (for TC-001 and TC-003).
- Before/after annotation values for TC-002 demonstrating the hash change.
- Node name lists before and after the test action for TC-001 and TC-003.

---

## 8. Suspension and Resumption Criteria

### 8.1 Suspension Criteria

- The HyperShift management cluster becomes unavailable or the HostedCluster enters a degraded state.
- The NodePool controller is crash-looping or not reconciling (operator pod not running).
- Infrastructure issues prevent node provisioning (cloud provider quota, networking).

### 8.2 Resumption Criteria

- The management cluster and HostedCluster are restored to a healthy, operational state.
- The HyperShift operator pod is running and reconciling NodePools.
- Any test NodePools created during a suspended test run are cleaned up before resuming.

---

## 9. Test Deliverables

| Deliverable | Description |
|-------------|-------------|
| This test plan | `plans/ocpstrat-9238.md` |
| Test execution log | Timestamped record of all commands executed and their outputs |
| Pass/fail summary | Per-test-case verdict with evidence references |
| Defect reports | Jira issues filed for any failures, linked to OCPSTRAT-3298 |

---

## 10. Test Environment

### 10.1 Required Infrastructure

| Component | Requirement |
|-----------|-------------|
| Management cluster | OpenShift cluster running the HyperShift operator with PR #8698 changes |
| HostedCluster | At least one operational HostedCluster with a default NodePool |
| Platform | AWS, Azure, or other supported platform (**not** KubeVirt for TC-002) |
| CLI tools | `oc` or `kubectl` with cluster-admin access to management cluster; kubeconfig for hosted cluster |

### 10.2 Platform Exclusions

- **KubeVirt:** TC-002 (spec-driven rollout) must be skipped on KubeVirt platform pending resolution of [CNV-38196](https://issues.redhat.com/browse/CNV-38196). TC-001 and TC-003 are platform-agnostic.

---

## 11. Responsibilities and Roles

| Role | Responsibility |
|------|---------------|
| Test author | Created this plan based on PR #8698 implementation |
| QE engineer | Executes the test cases, captures evidence, reports results |
| Feature developer | Provides clarification on expected behavior; reviews test results |
| QE lead | Reviews and approves this test plan; triages any failures |

---

## 12. Schedule and Milestones

| Milestone | Timing |
|-----------|--------|
| Test plan review | Before PR #8698 merges |
| Test environment provisioning | After PR #8698 merges and operator image is available |
| Test execution (TC-001, TC-002, TC-003) | Within one sprint of operator image availability |
| Results reporting | Within 2 business days of test execution |

---

## 13. Risks and Contingencies

| Risk | Impact | Mitigation |
|------|--------|------------|
| OCPSTRAT-9238 acceptance criteria unavailable | Cannot map test cases to formal acceptance criteria | Tests derived from PR implementation and e2e tests; re-map when Jira access is restored |
| PR #8698 not yet merged | Feature may change before merge | Re-review test plan if PR is substantially revised |
| NodePool provisioning timeouts | Test execution delayed | Use pre-existing NodePools for TC-001; allow 20+ minute timeouts for TC-002/TC-003 |
| KubeVirt platform exclusion | Reduced coverage | Track CNV-38196; add KubeVirt coverage when resolved |
| Annotation key changes before merge | Test commands use wrong annotation names | Verify annotation keys against merged code before execution |
| Flaky reconciliation timing | TC-003 may not see annotation re-seeded within 1 minute | Extend polling window to 2 minutes if needed; check operator logs for reconciliation evidence |

---

## 14. Approval and Exit Criteria

### 14.1 Approval

This test plan requires review and approval from the QE lead and the feature developer before test execution begins.

### 14.2 Exit Criteria

Testing is considered complete when:

1. All three test cases (TC-001, TC-002, TC-003) have been executed on at least one supported platform.
2. All pass criteria are met, or defects have been filed for any failures.
3. Test execution logs and evidence have been archived.
4. Results have been reported to the feature team.
5. Any filed defects have been linked to the feature Jira (OCPSTRAT-3298) and this test plan Jira (OCPSTRAT-9238, when accessible).
