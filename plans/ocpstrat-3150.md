# Test Plan: etcd Sharding for HyperShift Hosted Clusters

## 1. Test Plan Identifier

**ID:** TP-OCPSTRAT-3150-001
**Version:** 1.0
**Date:** 2026-09-07
**IEEE 829 Compliance:** This document follows the IEEE 829-2008 Standard for Software and System Test Documentation.

---

## 2. Introduction and Objectives

### 2.1 Purpose

This test plan defines the manual verification strategy for **etcd Sharding** in HyperShift hosted clusters. The feature distributes Kubernetes resources across multiple independent etcd clusters organized by resource type, addressing etcd performance limitations in very large hosted clusters or clusters with high resource churn.

The implementation spans two PRs:
- [PR #8705](https://github.com/openshift/hypershift/pull/8705) — Core etcd sharding by resource kind (merged)
- [PR #9146](https://github.com/openshift/hypershift/pull/9146) — Multi-shard etcd backup and restore support

### 2.2 Objectives

1. Verify that etcd shards are provisioned as independent StatefulSets, Services, PDBs, ServiceMonitors, and TLS certificates when configured in `spec.etcd.managed.shards`.
2. Verify that the kube-apiserver is configured with `--etcd-servers-overrides` to route specified resource types to the appropriate shard.
3. Verify that CEL validation rules enforce resource overlap prevention, shard immutability, name format, and storage constraints.
4. Verify that multi-shard backup captures snapshots from all PersistentVolume-backed shards and populates `ShardSnapshots` in `HCPEtcdBackupStatus`.
5. Verify that multi-shard restore injects `etcd-init` containers for shards with `restoreSnapshotURL` and aggregates the `EtcdSnapshotRestored` condition.
6. Verify backward compatibility: single-shard (unsharded) clusters continue to work; `SnapshotURL` in backup status is populated for the default shard.
7. Verify that the `EtcdSharding` feature gate controls visibility of shard-related API fields.

### 2.3 Acceptance Criteria Mapping

| Feature Capability | Test Case(s) |
|--------------------|--------------|
| Shard provisioning — StatefulSet, Service, PDB, ServiceMonitor, TLS per shard | TC-001 (§6.3) |
| KAS resource routing via `--etcd-servers-overrides` | TC-002 (§6.4) |
| Shard immutability — cannot add, remove, reorder, or rename shards after creation | TC-003 (§6.5) |
| CEL validation — resource overlap, name format, storage type constraints | TC-004 (§6.6) |
| Multi-shard backup — per-shard snapshots, EmptyDir skipped, backward compat | TC-005 (§6.7) |
| Multi-shard restore — init container injection, aggregate condition | TC-006 (§6.8) |
| Feature gate gating — fields hidden without `EtcdSharding` gate | TC-007 (§6.9) |
| Backward compatibility — unsharded cluster behavior unchanged | TC-008 (§6.10) |

---

## 3. Test Items and References

| Item | Reference |
|------|-----------|
| Sharding implementation PR | [openshift/hypershift#8705](https://github.com/openshift/hypershift/pull/8705) — "Add etcd sharding by resource kind support" |
| Backup/restore PR | [openshift/hypershift#9146](https://github.com/openshift/hypershift/pull/9146) — "Add multi-shard etcd backup and restore support" |
| Feature Jira | [OCPSTRAT-3150](https://issues.redhat.com/browse/OCPSTRAT-3150) — "etcd Sharding for HyperShift Hosted Clusters" |
| API: shard spec (managed) | `api/hypershift/v1beta1/hostedcluster_types.go` — `ManagedEtcdSpec.Shards`, `ManagedEtcdShardSpec`, `EtcdShardResource` |
| API: shard spec (unmanaged) | `api/hypershift/v1beta1/hostedcluster_types.go` — `UnmanagedEtcdSpec.Shards`, `UnmanagedEtcdShardSpec` |
| API: backup status | `api/hypershift/v1beta1/etcdbackup_types.go` — `HCPEtcdBackupStatus.ShardSnapshots`, `HCPEtcdShardSnapshot` |
| API: restore field | `api/hypershift/v1beta1/hostedcluster_types.go` — `ManagedEtcdShardSpec.RestoreSnapshotURL` |
| Feature gate | `EtcdSharding` (TechPreviewNoUpgrade) |
| CRD manifests | `zz_generated.featuregated-crd-manifests/hostedclusters.hypershift.openshift.io/EtcdSharding.yaml` |
| Controller: shard component | `control-plane-operator/controllers/hostedcontrolplane/v2/etcd/shard.go` |
| Controller: KAS params | `control-plane-operator/controllers/hostedcontrolplane/v2/kas/params.go`, `kas/deployment.go`, `kas/config.go` |
| Controller: backup reconciler | `hypershift-operator/controllers/etcdbackup/reconciler.go` |
| Controller: restore logic | `control-plane-operator/controllers/hostedcontrolplane/hostedcontrolplane_controller.go` |
| Controller: PKI (shard certs) | `control-plane-operator/controllers/hostedcontrolplane/pki/etcd.go` |
| Support: shard helpers | `support/etcd/shards.go` |
| etcd-upload CLI | `etcd-upload/etcdupload.go` |
| Unit tests: shard | `v2/etcd/shard_test.go`, `support/etcd/shards_test.go` |
| Unit tests: backup | `hypershift-operator/controllers/etcdbackup/reconciler_test.go` |
| Unit tests: restore | `control-plane-operator/controllers/hostedcontrolplane/hostedcontrolplane_controller_test.go` |
| Unit tests: KAS | `v2/kas/params_test.go` |
| CRD validation tests | `tests/hostedclusters.hypershift.openshift.io/featuregated.hostedclusters.etcdsharding.testsuite.yaml` |

### 3.1 Key API Types

| Type | Purpose |
|------|---------|
| `ManagedEtcdShardSpec` | Per-shard config: name, resources, storage, replicas, scheduling, restoreSnapshotURL |
| `EtcdShardResource` | Identifies a resource type (apiGroup + resource) to route to a shard |
| `EtcdShardSchedulingSpec` | Per-shard nodeSelector, tolerations, topology spread constraints |
| `HCPEtcdShardSnapshot` | Per-shard backup snapshot URL (name + snapshotURL) |
| `UnmanagedEtcdShardSpec` | Per-shard config for unmanaged etcd: name, resources, endpoint |

### 3.2 Key Constraints (from CEL validation)

| Rule | Description |
|------|-------------|
| `has(oldSelf.shards) == has(self.shards)` | Shards cannot be added or removed after creation |
| `self.size() == oldSelf.size()` | Shard count is immutable |
| `oldSelf.all(old, self.exists(cur, cur.name == old.name))` | Existing shards cannot be replaced |
| Resource overlap prevention | Resources must not overlap across shards |
| `restoreSnapshotURL` immutability | Once set, cannot be removed; value is immutable |
| EmptyDir + restoreSnapshotURL rejection | `restoreSnapshotURL` is not supported for EmptyDir shards |
| URL scheme validation | `restoreSnapshotURL` and `snapshotURL` must use `https` or `s3` scheme |

---

## 4. Features to Be Tested

1. **Managed etcd shard provisioning** — Creating a HostedCluster with `spec.etcd.managed.shards` entries causes independent StatefulSets, Services, PDBs, ServiceMonitors, and TLS secrets to be created for each shard in the hosted control plane namespace.
2. **KAS resource routing** — The kube-apiserver container args include `--etcd-servers-overrides` entries that map each shard's resources to its etcd endpoint. Resources written via the hosted cluster's API are stored in the correct shard.
3. **Shard immutability after creation** — Attempts to add, remove, reorder, rename shards, or change their storage type after initial HostedCluster creation are rejected by admission.
4. **CEL validation rules** — API server rejects: overlapping resources across shards; invalid shard names; invalid resource names; more than 10 shards; `restoreSnapshotURL` on EmptyDir shards; invalid URL schemes.
5. **Multi-shard backup** — An `HCPEtcdBackup` captures snapshots from the default shard and all PersistentVolume-backed named shards. EmptyDir-backed shards are skipped. `ShardSnapshots` in status contains per-shard snapshot URLs.
6. **Multi-shard restore** — Setting `restoreSnapshotURL` on shard specs causes `etcd-init` containers to be injected in the shard StatefulSet. The `EtcdSnapshotRestored` condition is aggregated across all shards.
7. **Feature gate enforcement** — Without the `EtcdSharding` feature gate enabled, the `shards` and `scheduling` fields are not present in the HostedCluster CRD schema.
8. **Backward compatibility** — A HostedCluster without shards (single default etcd) continues to operate normally. Backup status retains `snapshotURL` for the default shard alongside `shardSnapshots`.
9. **Per-shard storage options** — Shards can independently use `PersistentVolume` or `EmptyDir` storage types.
10. **Per-shard scheduling** — Each shard can have independent `nodeSelector`, `tolerations`, and `topologySpreadConstraints`.

---

## 5. Features Not to Be Tested

The following are explicitly out of scope for this manual test plan:

- **Unmanaged etcd sharding** — While the API supports `spec.etcd.unmanaged.shards`, unmanaged etcd is an external-cluster integration path. Manual verification of unmanaged shards is out of scope; API validation coverage via TC-004 is sufficient.
- **etcd-defrag behavior** — The `etcd-defrag` binary was updated to handle multiple instances, but defragmentation testing requires long-running load generation and is better covered by automated testing.
- **Performance and scale testing** — The feature targets 7,500+ node clusters; validating at-scale performance is outside manual testing scope.
- **CRD-backed / aggregated API resource routing** — By design, `--etcd-servers-overrides` only routes kube-apiserver built-in types. CRD and aggregated API resources (e.g., `openshift.io` groups) are NOT routed. This is a known kube-apiserver limitation, not a feature under test.
- **etcd-upload CLI internals** — The `--snapshot-dir` flag and JSON termination log changes are tested by unit tests. Manual CLI testing is out of scope.
- **Karpenter integration** — No Karpenter-specific changes in these PRs.
- **Build, deployment, or CI infrastructure** — This plan covers test execution only.
- **Multi-replica shard dynamics under failure** — Testing etcd quorum loss within individual shards requires fault injection infrastructure not available in manual testing.

---

## 6. Test Approach and Design

### 6.1 Approach

Each test scenario is executed manually against a running HyperShift management cluster. The tester uses `oc` or `kubectl` commands against the management cluster and the hosted cluster to verify outcomes through resource inspection, condition checks, and API validation. Test cases cover both positive (feature works as designed) and negative (invalid inputs are rejected) scenarios.

### 6.2 Common Prerequisites

- A running HyperShift management cluster with an operational HyperShift operator.
- The HyperShift operator version includes changes from both PR #8705 and PR #9146.
- The `EtcdSharding` feature gate is enabled (TechPreviewNoUpgrade or equivalent).
- The `HCPEtcdBackup` feature gate is enabled (for backup/restore test cases).
- `oc` / `kubectl` CLI access to the management cluster with sufficient permissions to create/patch/delete HostedCluster, HCPEtcdBackup objects, and read resources in hosted control plane namespaces.
- Access to a cloud provider (AWS, Azure, or GCP) for PersistentVolume-backed etcd storage.
- A kubeconfig for at least one hosted cluster's API (for resource routing verification).

---

### 6.3 Test Case TC-001: Managed Etcd Shard Provisioning

**Objective:** Confirm that creating a HostedCluster with etcd shards results in independent infrastructure resources (StatefulSet, Service, PDB, ServiceMonitor, TLS secrets) for each defined shard.

**Prerequisites:**
- Common prerequisites (§6.2) are satisfied.
- No pre-existing HostedCluster named `shard-test-001` exists.

**Steps:**

1. **Create a HostedCluster with two etcd shards:**

   Apply a HostedCluster manifest that includes etcd sharding configuration. The managed etcd section should define two shards — one for Events and one for Leases:
   ```yaml
   spec:
     etcd:
       managementType: Managed
       managed:
         storage:
           type: PersistentVolume
           persistentVolume:
             size: 8Gi
         shards:
           - name: events
             resources:
               - apiGroup: ""
                 resource: events
             storage:
               type: PersistentVolume
               persistentVolume:
                 size: 4Gi
             replicas: 1
           - name: leases
             resources:
               - apiGroup: coordination.k8s.io
                 resource: leases
             storage:
               type: EmptyDir
             replicas: 1
   ```

   Wait for the HostedCluster to reach the `Available` condition:
   ```
   oc wait hostedcluster shard-test-001 -n <namespace> --for=condition=Available --timeout=30m
   ```

2. **Identify the hosted control plane namespace:**
   ```
   HC_NS=$(oc get hostedcluster shard-test-001 -n <namespace> -o jsonpath='{.status.controlPlaneNamespace}')
   ```

3. **Verify StatefulSets exist for each shard:**
   ```
   oc get statefulset -n "$HC_NS" -l app=etcd
   ```
   Expected: Three StatefulSets are present:
   - `etcd` (default shard)
   - `etcd-events` (events shard)
   - `etcd-leases` (leases shard)

4. **Verify Services exist for each shard:**
   ```
   oc get service -n "$HC_NS" | grep etcd
   ```
   Expected: Services for `etcd-events` and `etcd-leases` exist alongside the default `etcd` services (client and discovery).

5. **Verify PodDisruptionBudgets exist for each shard:**
   ```
   oc get pdb -n "$HC_NS" | grep etcd
   ```
   Expected: PDBs for `etcd-events` and `etcd-leases` exist alongside the default `etcd` PDB.

6. **Verify ServiceMonitors exist for each shard:**
   ```
   oc get servicemonitor -n "$HC_NS" | grep etcd
   ```
   Expected: ServiceMonitors for `etcd-events` and `etcd-leases` exist alongside the default.

7. **Verify TLS secrets exist for each shard:**
   ```
   oc get secret -n "$HC_NS" | grep -E 'etcd.*(peer|client|server)'
   ```
   Expected: Peer, client, and server certificate secrets exist for each shard (e.g., `etcd-events-peer-tls`, `etcd-leases-peer-tls`).

8. **Verify storage types are honored:**

   Check the events shard uses PersistentVolume:
   ```
   oc get statefulset etcd-events -n "$HC_NS" -o jsonpath='{.spec.volumeClaimTemplates[0].spec.resources.requests.storage}'
   ```
   Expected: `4Gi`.

   Check the leases shard uses EmptyDir:
   ```
   oc get statefulset etcd-leases -n "$HC_NS" -o jsonpath='{.spec.template.spec.volumes}' | grep emptyDir
   ```
   Expected: An `emptyDir` volume is present (no PVC template).

9. **Verify shard pods are running:**
   ```
   oc get pods -n "$HC_NS" -l app=etcd
   ```
   Expected: Pods for all three StatefulSets are in `Running` state.

**Pass criteria:** All shard infrastructure resources (StatefulSet, Service, PDB, ServiceMonitor, TLS secrets) are created with correct naming, storage types are honored, and pods are running.

**Fail criteria:** Any shard-specific resource is missing; storage types do not match configuration; pods are not running.

---

### 6.4 Test Case TC-002: KAS Resource Routing via etcd-servers-overrides

**Objective:** Confirm that the kube-apiserver is configured with `--etcd-servers-overrides` to route specified resource types to the appropriate etcd shard, and that resources are actually stored in the correct shard.

**Prerequisites:**
- TC-001 has been executed and the HostedCluster `shard-test-001` is available with Events and Leases shards.

**Steps:**

1. **Verify the KAS deployment includes --etcd-servers-overrides:**
   ```
   oc get deployment kube-apiserver -n "$HC_NS" -o jsonpath='{.spec.template.spec.containers[?(@.name=="kube-apiserver")].command}' | tr ',' '\n' | grep etcd-servers-overrides
   ```
   Expected: Output contains `--etcd-servers-overrides` with entries for:
   - `/events#https://etcd-events-client:2379` (or equivalent service URL)
   - `coordination.k8s.io/leases#https://etcd-leases-client:2379` (or equivalent service URL)

2. **Create an Event in the hosted cluster and verify it was stored:**

   Using the hosted cluster's kubeconfig:
   ```
   oc --kubeconfig=<hosted-kubeconfig> create namespace shard-routing-test
   oc --kubeconfig=<hosted-kubeconfig> run test-pod --image=busybox --restart=Never -n shard-routing-test -- sleep 3600
   ```

   Wait briefly for Events to be generated:
   ```
   oc --kubeconfig=<hosted-kubeconfig> get events -n shard-routing-test
   ```
   Expected: Events exist (e.g., pod scheduling, pulling image).

3. **Verify the events shard contains data:**
   ```
   oc exec -n "$HC_NS" etcd-events-0 -c etcd -- etcdctl \
     --cacert /etc/etcd/tls/etcd-ca/ca.crt \
     --cert /etc/etcd/tls/client/tls.crt \
     --key /etc/etcd/tls/client/tls.key \
     --endpoints https://localhost:2379 \
     get /kubernetes.io/events/ --prefix --keys-only --limit=5
   ```
   Expected: Keys with the prefix `/kubernetes.io/events/` are present in the events shard.

4. **Verify the default shard does NOT contain events:**
   ```
   oc exec -n "$HC_NS" etcd-0 -c etcd -- etcdctl \
     --cacert /etc/etcd/tls/etcd-ca/ca.crt \
     --cert /etc/etcd/tls/client/tls.crt \
     --key /etc/etcd/tls/client/tls.key \
     --endpoints https://localhost:2379 \
     get /kubernetes.io/events/ --prefix --keys-only --limit=5
   ```
   Expected: No keys with the `/kubernetes.io/events/` prefix are found in the default shard.

5. **Verify non-routed resources remain in the default shard:**
   ```
   oc exec -n "$HC_NS" etcd-0 -c etcd -- etcdctl \
     --cacert /etc/etcd/tls/etcd-ca/ca.crt \
     --cert /etc/etcd/tls/client/tls.crt \
     --key /etc/etcd/tls/client/tls.key \
     --endpoints https://localhost:2379 \
     get /kubernetes.io/namespaces/ --prefix --keys-only --limit=5
   ```
   Expected: Namespace keys are present in the default shard (namespaces are not routed to any named shard).

**Pass criteria:** KAS `--etcd-servers-overrides` flag contains correct routing entries; events are stored in the events shard and absent from the default shard; non-routed resources remain in the default shard.

**Fail criteria:** Routing flag is missing or incorrect; events are found in the default shard; events are absent from the events shard.

---

### 6.5 Test Case TC-003: Shard Immutability After Creation

**Objective:** Confirm that attempts to modify the shard list (add, remove, reorder, or rename shards) or change shard storage type after HostedCluster creation are rejected by API admission.

**Prerequisites:**
- TC-001 has been executed and `shard-test-001` exists with two shards (`events`, `leases`).

**Steps:**

1. **Attempt to add a third shard:**
   ```
   oc patch hostedcluster shard-test-001 -n <namespace> --type=json \
     -p '[{"op":"add","path":"/spec/etcd/managed/shards/-","value":{"name":"pods","resources":[{"apiGroup":"","resource":"pods"}],"storage":{"type":"EmptyDir"},"replicas":1}}]'
   ```
   Expected: The request is **rejected** with a validation error containing `"shards cannot be added or removed after creation"`.

2. **Attempt to remove an existing shard:**
   ```
   oc patch hostedcluster shard-test-001 -n <namespace> --type=json \
     -p '[{"op":"remove","path":"/spec/etcd/managed/shards/1"}]'
   ```
   Expected: The request is **rejected** with `"shards cannot be added or removed after creation"`.

3. **Attempt to rename a shard:**
   ```
   oc patch hostedcluster shard-test-001 -n <namespace> --type=json \
     -p '[{"op":"replace","path":"/spec/etcd/managed/shards/0/name","value":"renamed-events"}]'
   ```
   Expected: The request is **rejected** with `"existing shards cannot be replaced"`.

4. **Attempt to change a shard's storage type:**
   ```
   oc patch hostedcluster shard-test-001 -n <namespace> --type=json \
     -p '[{"op":"replace","path":"/spec/etcd/managed/shards/0/storage/type","value":"EmptyDir"}]'
   ```
   Expected: The request is **rejected** with `"storage cannot be added or removed after creation"`.

5. **Attempt to change a shard's resources:**
   ```
   oc patch hostedcluster shard-test-001 -n <namespace> --type=json \
     -p '[{"op":"replace","path":"/spec/etcd/managed/shards/0/resources","value":[{"apiGroup":"","resource":"configmaps"}]}]'
   ```
   Expected: The request is **rejected** with `"resources are immutable"`.

**Pass criteria:** All five mutation attempts are rejected with appropriate validation error messages.

**Fail criteria:** Any mutation attempt is accepted by the API server.

---

### 6.6 Test Case TC-004: CEL Validation Rules

**Objective:** Confirm that the API server correctly validates shard configuration: resource overlap, name format, shard limits, EmptyDir+restoreSnapshotURL rejection, and URL scheme validation.

**Prerequisites:**
- Common prerequisites (§6.2) are satisfied.
- No pre-existing HostedCluster named `cel-test-*` exists.

**Steps:**

1. **Reject overlapping resources across shards:**

   Attempt to create a HostedCluster where two shards route the same resource:
   ```yaml
   shards:
     - name: shard-a
       resources:
         - apiGroup: ""
           resource: events
       storage:
         type: EmptyDir
       replicas: 1
     - name: shard-b
       resources:
         - apiGroup: ""
           resource: events
       storage:
         type: EmptyDir
       replicas: 1
   ```
   Expected: Creation is **rejected** with `"resources must not overlap across shards"`.

2. **Reject more than 10 shards:**

   Attempt to create a HostedCluster with 11 shard entries.

   Expected: Creation is **rejected** with a MaxItems validation error.

3. **Reject restoreSnapshotURL on EmptyDir shard:**

   Attempt to create a HostedCluster with:
   ```yaml
   shards:
     - name: events
       resources:
         - apiGroup: ""
           resource: events
       storage:
         type: EmptyDir
       replicas: 1
       restoreSnapshotURL: "https://storage.example.com/snapshot.db"
   ```
   Expected: Creation is **rejected** with `"restoreSnapshotURL is not supported for EmptyDir shards"`.

4. **Reject invalid restoreSnapshotURL scheme:**

   Attempt to create a HostedCluster with a shard that has:
   ```yaml
   restoreSnapshotURL: "http://insecure.example.com/snapshot.db"
   ```
   Expected: Creation is **rejected** with `"restoreSnapshotURL must be a valid URL with scheme https or s3"`.

5. **Accept valid shard configuration:**

   Create a HostedCluster with a single valid shard (non-overlapping resources, valid name, PV storage):
   ```yaml
   shards:
     - name: events
       resources:
         - apiGroup: ""
           resource: events
       storage:
         type: PersistentVolume
         persistentVolume:
           size: 4Gi
       replicas: 1
   ```
   Expected: Creation is **accepted** without validation errors.

**Pass criteria:** All four negative cases are rejected with correct messages; the positive case is accepted.

**Fail criteria:** Any negative case is accepted; the positive case is rejected.

---

### 6.7 Test Case TC-005: Multi-Shard Backup

**Objective:** Confirm that an `HCPEtcdBackup` captures snapshots from all PersistentVolume-backed shards, skips EmptyDir-backed shards, populates `ShardSnapshots` in status, and maintains backward compatibility via `SnapshotURL`.

**Prerequisites:**
- TC-001 has been executed and `shard-test-001` is available with:
  - `events` shard (PersistentVolume)
  - `leases` shard (EmptyDir)
- The `HCPEtcdBackup` feature gate is enabled.
- Cloud storage is configured for backup snapshots (e.g., S3 bucket).

**Steps:**

1. **Create an HCPEtcdBackup resource:**
   ```yaml
   apiVersion: hypershift.openshift.io/v1beta1
   kind: HCPEtcdBackup
   metadata:
     name: shard-backup-test
     namespace: <hc-namespace>
   spec:
     hostedClusterRef:
       name: shard-test-001
   ```
   Apply:
   ```
   oc apply -f <backup-manifest>.yaml
   ```

2. **Wait for the backup to complete:**
   ```
   oc wait hcpetcdbackup shard-backup-test -n <hc-namespace> \
     --for=condition=BackupCompleted=True --timeout=15m
   ```

3. **Verify ShardSnapshots contains entries for PV-backed shards:**
   ```
   oc get hcpetcdbackup shard-backup-test -n <hc-namespace> \
     -o jsonpath='{.status.shardSnapshots[*].name}'
   ```
   Expected: Output contains both `etcd` (the default shard) and `etcd-events` (the PV-backed named shard).

4. **Verify EmptyDir-backed shard is NOT in ShardSnapshots:**
   ```
   oc get hcpetcdbackup shard-backup-test -n <hc-namespace> \
     -o jsonpath='{.status.shardSnapshots[*].name}' | grep -c leases
   ```
   Expected: `0` — the leases shard (EmptyDir) is not included.

5. **Verify each ShardSnapshot has a valid snapshotURL:**
   ```
   oc get hcpetcdbackup shard-backup-test -n <hc-namespace> \
     -o jsonpath='{range .status.shardSnapshots[*]}{.name}: {.snapshotURL}{"\n"}{end}'
   ```
   Expected: Each entry has a non-empty URL with `https://` or `s3://` scheme.

6. **Verify backward-compatible SnapshotURL is populated:**
   ```
   oc get hcpetcdbackup shard-backup-test -n <hc-namespace> \
     -o jsonpath='{.status.snapshotURL}'
   ```
   Expected: Non-empty URL matching the default shard's snapshot (same value as the `etcd` entry in `shardSnapshots`).

**Pass criteria:** `ShardSnapshots` contains entries for PV-backed shards only; each has a valid snapshot URL; `SnapshotURL` (top-level) matches the default shard entry; EmptyDir shard is excluded.

**Fail criteria:** `ShardSnapshots` is empty; EmptyDir shard is included; `SnapshotURL` is empty or does not match the default shard.

---

### 6.8 Test Case TC-006: Multi-Shard Restore

**Objective:** Confirm that setting `restoreSnapshotURL` on shard specs causes the shard StatefulSet to include an `etcd-init` container for snapshot restore, and the `EtcdSnapshotRestored` condition is aggregated across all shards.

**Prerequisites:**
- A valid backup has been completed (TC-005) and snapshot URLs are available.
- A new HostedCluster will be created with restore configuration.

**Steps:**

1. **Create a HostedCluster with restoreSnapshotURL on the default shard and a named shard:**

   Use the snapshot URLs captured from TC-005. Configure the HostedCluster with:
   ```yaml
   spec:
     etcd:
       managementType: Managed
       managed:
         storage:
           type: PersistentVolume
           persistentVolume:
             size: 8Gi
           restoreSnapshotURL: "<default-shard-snapshot-url>"
         shards:
           - name: events
             resources:
               - apiGroup: ""
                 resource: events
             storage:
               type: PersistentVolume
               persistentVolume:
                 size: 4Gi
             replicas: 1
             restoreSnapshotURL: "<events-shard-snapshot-url>"
   ```

2. **Identify the hosted control plane namespace:**
   ```
   HC_NS=$(oc get hostedcluster shard-restore-test -n <namespace> -o jsonpath='{.status.controlPlaneNamespace}')
   ```

3. **Verify the events shard StatefulSet includes an etcd-init container:**
   ```
   oc get statefulset etcd-events -n "$HC_NS" \
     -o jsonpath='{.spec.template.spec.initContainers[*].name}'
   ```
   Expected: An `etcd-init` container is present in the init container list.

4. **Verify the etcd-init container references the correct snapshot URL:**
   ```
   oc get statefulset etcd-events -n "$HC_NS" \
     -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="etcd-init")].env}' | grep -o 'https://[^ ]*\|s3://[^ ]*'
   ```
   Expected: The snapshot URL matches the `restoreSnapshotURL` from the shard spec.

5. **Wait for the HostedCluster to become available:**
   ```
   oc wait hostedcluster shard-restore-test -n <namespace> \
     --for=condition=Available --timeout=30m
   ```

6. **Verify the EtcdSnapshotRestored condition is True:**
   ```
   oc get hostedcluster shard-restore-test -n <namespace> \
     -o jsonpath='{.status.conditions[?(@.type=="EtcdSnapshotRestored")].status}'
   ```
   Expected: `True`.

7. **Verify restoreSnapshotURL immutability:**
   ```
   oc patch hostedcluster shard-restore-test -n <namespace> --type=json \
     -p '[{"op":"replace","path":"/spec/etcd/managed/shards/0/restoreSnapshotURL","value":"https://other.example.com/new-snapshot.db"}]'
   ```
   Expected: The request is **rejected** with `"restoreSnapshotURL is immutable"`.

8. **Verify restoreSnapshotURL cannot be removed once set:**
   ```
   oc patch hostedcluster shard-restore-test -n <namespace> --type=json \
     -p '[{"op":"remove","path":"/spec/etcd/managed/shards/0/restoreSnapshotURL"}]'
   ```
   Expected: The request is **rejected** with `"restoreSnapshotURL cannot be removed once set"`.

**Pass criteria:** `etcd-init` container is present with correct snapshot URL; `EtcdSnapshotRestored` condition is `True`; `restoreSnapshotURL` is immutable and cannot be removed.

**Fail criteria:** Init container is missing; condition is not `True`; immutability or removal guard is not enforced.

---

### 6.9 Test Case TC-007: Feature Gate Enforcement

**Objective:** Confirm that the `EtcdSharding` feature gate controls visibility of shard-related API fields.

**Prerequisites:**
- A HyperShift management cluster where the `EtcdSharding` feature gate is **not** enabled (Default feature set without TechPreviewNoUpgrade).

**Steps:**

1. **Inspect the HostedCluster CRD for the shards field:**
   ```
   oc get crd hostedclusters.hypershift.openshift.io -o json | \
     python3 -c "
   import json, sys
   crd = json.load(sys.stdin)
   schema = crd['spec']['versions'][0]['schema']['openAPIV3Schema']
   etcd_managed = schema['properties']['spec']['properties']['etcd']['properties']['managed']
   print('shards' in etcd_managed.get('properties', {}))
   "
   ```
   Expected: `False` — the `shards` field is not present in the CRD schema when the feature gate is disabled.

2. **Inspect the HostedCluster CRD for the scheduling field on managed etcd:**
   ```
   oc get crd hostedclusters.hypershift.openshift.io -o json | \
     python3 -c "
   import json, sys
   crd = json.load(sys.stdin)
   schema = crd['spec']['versions'][0]['schema']['openAPIV3Schema']
   etcd_managed = schema['properties']['spec']['properties']['etcd']['properties']['managed']
   print('scheduling' in etcd_managed.get('properties', {}))
   "
   ```
   Expected: `False` — the `scheduling` field is not present without the feature gate.

3. **Attempt to create a HostedCluster with shards (gate disabled):**

   Apply a HostedCluster manifest that includes `spec.etcd.managed.shards`.

   Expected: The `shards` field is silently dropped (unknown field pruning) or the request is rejected, depending on CRD strictness.

**Pass criteria:** Shard-related fields are not exposed in the CRD schema without the feature gate; attempts to use them are ineffective.

**Fail criteria:** Shard fields are visible or functional without the feature gate enabled.

---

### 6.10 Test Case TC-008: Backward Compatibility — Unsharded Cluster

**Objective:** Confirm that a HostedCluster created without any shard configuration continues to operate with a single default etcd and that backup produces a single-entry `ShardSnapshots` list alongside the traditional `SnapshotURL`.

**Prerequisites:**
- Common prerequisites (§6.2) are satisfied.
- A HostedCluster exists without any `spec.etcd.managed.shards` configuration (standard single-etcd setup).

**Steps:**

1. **Verify only the default etcd StatefulSet exists:**
   ```
   oc get statefulset -n "$HC_NS" -l app=etcd --no-headers
   ```
   Expected: Only a single StatefulSet named `etcd` is listed.

2. **Verify no --etcd-servers-overrides flag is present:**
   ```
   oc get deployment kube-apiserver -n "$HC_NS" \
     -o jsonpath='{.spec.template.spec.containers[?(@.name=="kube-apiserver")].command}' | grep etcd-servers-overrides
   ```
   Expected: No output (the flag is absent when there are no shards).

3. **Create a backup of the unsharded cluster:**
   ```
   oc apply -f <backup-manifest-for-unsharded>.yaml
   oc wait hcpetcdbackup unsharded-backup-test -n <hc-namespace> \
     --for=condition=BackupCompleted=True --timeout=15m
   ```

4. **Verify ShardSnapshots contains a single entry for the default shard:**
   ```
   oc get hcpetcdbackup unsharded-backup-test -n <hc-namespace> \
     -o jsonpath='{.status.shardSnapshots}'
   ```
   Expected: A single entry with `name: "etcd"` and a valid `snapshotURL`.

5. **Verify SnapshotURL matches the default shard entry:**
   ```
   oc get hcpetcdbackup unsharded-backup-test -n <hc-namespace> \
     -o jsonpath='{.status.snapshotURL}'
   ```
   Expected: Same URL as the `etcd` entry in `shardSnapshots`.

**Pass criteria:** Unsharded cluster operates with a single etcd; backup produces both `ShardSnapshots` (single entry) and `SnapshotURL`; no `--etcd-servers-overrides` flag is present.

**Fail criteria:** Extra StatefulSets appear; `--etcd-servers-overrides` is present; backup status is missing or inconsistent.

---

## 7. Pass/Fail Criteria

### 7.1 Overall Pass

All eight test cases (TC-001 through TC-008) pass their individual criteria as defined in §6.3–§6.10.

### 7.2 Overall Fail

Any single test case fails its criteria.

### 7.3 Evidence Requirements

For each test case, the tester must capture:
- Terminal output or screenshots of all `oc get` / `oc patch` / `oc exec` commands showing resource states, annotation values, condition statuses, and validation error messages.
- Before/after comparisons where applicable (e.g., shard lists, annotation values).
- Timestamped observation records for provisioning and backup completion waits.
- etcdctl key listings for routing verification (TC-002).

---

## 8. Suspension and Resumption Criteria

### 8.1 Suspension Criteria

- The HyperShift management cluster becomes unavailable or the HostedCluster enters a degraded state unrelated to the feature under test.
- The HyperShift operator is crash-looping or not reconciling.
- Cloud provider infrastructure issues prevent etcd PersistentVolume provisioning or backup storage access.
- The `EtcdSharding` or `HCPEtcdBackup` feature gate is unexpectedly disabled or the operator version does not include the required PRs.

### 8.2 Resumption Criteria

- The management cluster and HostedCluster are restored to a healthy, operational state.
- The HyperShift operator pod is running and reconciling.
- Any test HostedClusters created during a suspended test run are cleaned up before resuming.
- Feature gates are confirmed enabled and the correct operator version is deployed.

---

## 9. Test Deliverables

| Deliverable | Description |
|-------------|-------------|
| This test plan | `plans/ocpstrat-3150.md` |
| Test execution log | Timestamped record of all commands executed and their outputs |
| Pass/fail summary | Per-test-case verdict with evidence references |
| Defect reports | Jira issues filed for any failures, linked to [OCPSTRAT-3150](https://issues.redhat.com/browse/OCPSTRAT-3150) |

---

## 10. Test Environment

### 10.1 Required Infrastructure

| Component | Requirement |
|-----------|-------------|
| Management cluster | OpenShift cluster running the HyperShift operator with PR #8705 and PR #9146 changes |
| Feature gates | `EtcdSharding` (TechPreviewNoUpgrade) and `HCPEtcdBackup` enabled |
| HostedCluster | At least one operational HostedCluster (created as part of testing) |
| Platform | AWS, Azure, or GCP with PersistentVolume support for etcd storage |
| Backup storage | S3 bucket or equivalent cloud storage configured for HCPEtcdBackup |
| CLI tools | `oc` or `kubectl` with cluster-admin access to management cluster; kubeconfig for hosted cluster |
| etcdctl | Available within etcd pods (used in TC-002 for key inspection) |

### 10.2 Feature Gate Configuration

The following feature gates must be enabled on the HyperShift operator:
- `EtcdSharding` — Required for all test cases
- `HCPEtcdBackup` — Required for TC-005, TC-006, TC-008

For TC-007 (feature gate enforcement), a separate cluster or operator configuration with `EtcdSharding` **disabled** is required.

---

## 11. Responsibilities and Roles

| Role | Responsibility |
|------|---------------|
| Test author | Created this plan based on PR #8705 and PR #9146 implementation |
| QE engineer | Executes the test cases, captures evidence, reports results |
| Feature developer | Provides clarification on expected behavior; reviews test results |
| QE lead | Reviews and approves this test plan; triages any failures |

---

## 12. Schedule and Milestones

| Milestone | Timing |
|-----------|--------|
| Test plan review | Before PR #9146 merges |
| Test environment provisioning | After both PRs merge and operator image is available |
| Test execution — core sharding (TC-001 through TC-004, TC-007, TC-008) | Within one sprint of operator image availability |
| Test execution — backup/restore (TC-005, TC-006) | After PR #9146 merges and operator image includes both PRs |
| Results reporting | Within 2 business days of test execution |

---

## 13. Risks and Contingencies

| Risk | Impact | Mitigation |
|------|--------|------------|
| PR #9146 not yet merged | Backup/restore test cases (TC-005, TC-006) cannot execute | Execute core sharding tests (TC-001–TC-004, TC-007, TC-008) first; defer backup/restore tests until merge |
| Feature gate behavior changes | CRD field visibility may differ | Verify CRD schema against actual operator deployment before executing TC-007 |
| etcdctl not available in shard pods | TC-002 routing verification impossible | Use alternative verification: check KAS logs for routing configuration, or create resources and verify via API behavior |
| Cloud storage configuration issues | Backup tests (TC-005, TC-006) blocked | Pre-validate backup storage access by running a simple backup on an unsharded cluster first |
| PersistentVolume provisioning delays | HostedCluster creation timeouts | Use pre-existing storage classes; extend wait timeouts to 30+ minutes |
| CEL validation rule changes | Error messages in TC-003, TC-004 may differ | Verify expected error messages against the merged CRD manifests before execution |
| Shard naming convention changes | Resource names in TC-001 may differ (e.g., `etcd-events` vs `etcd-shard-events`) | Inspect actual StatefulSet names after HostedCluster creation; adapt commands accordingly |
| KAS `--etcd-servers-overrides` format changes | TC-002 grep patterns may not match | Inspect the full KAS command and adapt the grep pattern to the actual flag format |

---

## 14. Approval and Exit Criteria

### 14.1 Approval

This test plan requires review and approval from the QE lead and the feature developer before test execution begins.

### 14.2 Exit Criteria

Testing is considered complete when:

1. All eight test cases (TC-001 through TC-008) have been executed on at least one supported platform.
2. All pass criteria are met, or defects have been filed for any failures.
3. Test execution logs and evidence have been archived.
4. Results have been reported to the feature team.
5. Any filed defects have been linked to the feature Jira ([OCPSTRAT-3150](https://issues.redhat.com/browse/OCPSTRAT-3150)).
