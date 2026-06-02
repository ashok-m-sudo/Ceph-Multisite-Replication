# Deployment Guide

This guide walks through bringing up the two-node Ceph multisite prototype end-to-end. It assumes the [prerequisites](../README.md#-prerequisites) are met — in particular, that a healthy Rook-Ceph cluster is already running in the `rook-ceph` namespace on both `primary-node` and `secondary-node`.

All commands are run from the operator workstation against the respective cluster's `kubectl` context. Where the same command runs on both nodes, the heading makes that explicit.

---

## Phase 1 — Primary Site (Site A, `zone-a` master)

### Step 1. Clone the Rook examples

The multisite CRDs ship as YAML templates in the Rook repository. Pin to the same version as your installed Rook operator.

```bash
git clone --single-branch --branch v1.16.5 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f csi-operator.yaml
kubectl create -f operator.yaml -f cluster.yaml
```

### Step 2. Apply the multisite object store manifest

The `object-multisite.yaml` manifest declares the `CephObjectRealm`, `CephObjectZoneGroup`, `CephObjectZone`, and `CephObjectStore` resources for the master zone. Edit it to match this PoC:

- `CephObjectRealm` → `realm-a`
- `CephObjectZoneGroup` → `zonegroup-a` (referencing `realm-a`)
- `CephObjectZone` → `zone-a` (referencing `zonegroup-a`)
- `CephObjectStore` → bound to `zone-a`

```bash
vi object-multisite.yaml
kubectl create -f object-multisite.yaml
```

> 💡 A sanitized reference copy lives at [`configs/primary/object-multisite.yaml`](../configs/primary/object-multisite.yaml).

Wait for the RGW pod to reach `Running`:

```bash
kubectl get pod -n rook-ceph -w
```

Expect a pod with a name like `rook-ceph-rgw-multisite-store-a-*`.

### Step 3. Expose the RGW endpoint externally

Create a `LoadBalancer` / external-IP service so the secondary site can reach the master RGW on `10.0.0.10:7474`. Edit `rgw-external.yaml` so the `selector` matches the `multisite-store` CephObjectStore name and apply it:

```bash
vi rgw-external.yaml
kubectl create -f rgw-external.yaml
```

### Step 4. Attach the external IP to the service

Confirm the service exists, then edit it to add `10.0.0.10` as the external IP:

```bash
kubectl get svc -n rook-ceph
kubectl edit svc rook-ceph-rgw-multisite-store-external -n rook-ceph
```

In the editor, set:

```yaml
spec:
  externalIPs:
    - 10.0.0.10
```

Save and exit.

### Step 5. Verify L4 connectivity

From a host outside the cluster (or from `secondary-node` once you have it provisioned), confirm the RGW endpoint is reachable:

```bash
telnet 10.0.0.10 7474
```

A successful TCP handshake is sufficient at this stage.

### Step 6. Confirm Ceph multisite resources are healthy

```bash
kubectl get cephobjectrealm     -n rook-ceph
kubectl get cephobjectzonegroup -n rook-ceph
kubectl get cephobjectzone      -n rook-ceph
kubectl get cephobjectstore     -n rook-ceph
```

All four resources should report `Connected` / `Ready` phases.

### Step 7. Create the system user

The secondary site's `ceph-radosgw` daemon must authenticate to the master before it can pull realm metadata. Create a dedicated system user inside the toolbox pod (or any pod with the `radosgw-admin` binary):

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- bash

radosgw-admin user create \
  --uid="synchronization-user" \
  --display-name="Synchronization User" \
  --system
```

**📌 Record the `access_key` and `secret_key` from the output** — they are needed for both the master zone modification (below) and the secondary site's realm pull (Phase 2).

### Step 8. Attach the system user keys to the master zone

```bash
radosgw-admin zone modify \
  --rgw-zone=zone-a \
  --access-key=<ACCESS_KEY_FROM_STEP_7> \
  --secret=<SECRET_KEY_FROM_STEP_7>
```

### Step 9. Commit the period

Every change to realm/zonegroup/zone configuration must be committed via a new period. This is what propagates the master's view of the world to all secondaries that subsequently pull it.

```bash
radosgw-admin period update --commit
```

Site A is now ready to serve as the multisite master.

---

## Phase 2 — Secondary Site (Site B, `zone-b` replica)

All commands in this phase target the `secondary-node` cluster's `kubectl` context.

### Step 1. Author the pull-realm manifest

On `secondary-node`, create the manifest that defines `zone-b` and tells Rook to pull `realm-a` from the master endpoint.

```bash
vi object-multisite-pull-realm.yaml
```

Key fields to set:

- `CephObjectRealm.spec.pull.endpoint` → `http://10.0.0.10:7474`
- The realm's `accessKey` / `secretKey` Secret values → the system user keys from Phase 1, Step 7
- `CephObjectZone` → `zone-b`, referencing `zonegroup-a` and `realm-a`
- `CephObjectStore` → bound to `zone-b`

> 💡 A sanitized reference copy lives at [`configs/secondary/object-multisite-pull-realm.yaml`](../configs/secondary/object-multisite-pull-realm.yaml).

### Step 2. Apply the manifest

```bash
kubectl create -f object-multisite-pull-realm.yaml
```

Watch for the RGW pod on Site B to come up:

```bash
kubectl get pod -n rook-ceph -w
```

### Step 3. Expose the secondary RGW endpoint

```bash
vi rgw-external.yaml      # set selector to the zone-b CephObjectStore name
kubectl create -f rgw-external.yaml
```

Then attach the external IP:

```bash
kubectl get svc -n rook-ceph
kubectl edit svc rook-ceph-rgw-<zone-b-store-name>-external -n rook-ceph
# add externalIPs: [10.0.0.20]
```

### Step 4. Verify L4 connectivity

```bash
telnet 10.0.0.20 7474
```

### Step 5. Confirm Ceph multisite resources on the secondary

```bash
kubectl get cephobjectrealm     -n rook-ceph
kubectl get cephobjectzonegroup -n rook-ceph
kubectl get cephobjectzone      -n rook-ceph
kubectl get cephobjectstore     -n rook-ceph
radosgw-admin sync status
```

If `sync status` reports healthy preparation, you can skip to [verification](verification.md). If it errors (typically due to the realm not being pulled cleanly during pod startup), use the manual fallback below.

---

## Phase 3 — Manual Reconciliation (only if Phase 2 sync fails)

In some lab conditions the operator-driven realm pull can race with the RGW pod startup. If `radosgw-admin sync status` reports `failed to fetch realm` or similar, drive the pull manually from inside the toolbox on `secondary-node`:

### Step 1. Pull the realm directly

```bash
radosgw-admin realm pull \
  --url=http://10.0.0.10:7474 \
  --access-key=<ACCESS_KEY_FROM_PHASE_1_STEP_7> \
  --secret=<SECRET_KEY_FROM_PHASE_1_STEP_7>
```

### Step 2. Default the realm (only if it is the only one)

```bash
radosgw-admin realm default --rgw-realm=realm-a
```

### Step 3. Register the secondary zone's endpoint

This tells the master zonegroup where to send sync requests for `zone-b`.

```bash
radosgw-admin zone modify \
  --rgw-zonegroup=zonegroup-a \
  --rgw-zone=zone-b \
  --access-key=<ACCESS_KEY> \
  --secret=<SECRET_KEY> \
  --endpoints=http://10.0.0.20:7474
```

### Step 4. Commit the new period

```bash
radosgw-admin period update --commit
radosgw-admin sync status
```

A healthy `sync status` shows the secondary `zone-b` polling `zone-a` for data and metadata, with no per-shard backlog stuck non-decreasing.

---

## Done

Proceed to [`verification.md`](verification.md) to validate end-to-end replication.