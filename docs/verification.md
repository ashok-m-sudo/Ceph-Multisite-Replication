# Verification & Replication Tests

This document covers how to prove the multisite setup is actually replicating — not just configured. Run these checks after completing [`deployment-guide.md`](deployment-guide.md).

The tests progress from cheapest to most exhaustive:

1. **Static checks** — confirm metadata and zone roles are correct.
2. **Sync status** — confirm the sync workers are alive and not backlogged.
3. **Bucket replication** — create a bucket on one side, observe it on the other.
4. **Object replication** — upload payloads, confirm content and integrity converge.
5. **Failure-mode probes** — error lists and pod logs for when things go wrong.

Throughout, commands prefixed with `# on primary-node` or `# on secondary-node` should be executed inside the Rook toolbox pod on that cluster:

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- bash
```

---

## 1. Static Configuration Checks

### Confirm the realm, zonegroup, and zone identities

Both sides should agree on `realm-a` and `zonegroup-a`. Only the local zone differs.

```bash
# on primary-node
radosgw-admin realm list
radosgw-admin zonegroup list
radosgw-admin zone list
```

Expected on `primary-node`:

```
realm-a       (default)
zonegroup-a   (default, master)
zone-a        (default, master)
```

Expected on `secondary-node`:

```
realm-a       (default)
zonegroup-a   (default, master)
zone-b        (default)
```

### Confirm the period is in sync

The `realm_epoch` and `id` should match on both sides:

```bash
radosgw-admin period get | grep -E '"id"|"epoch"|"realm"'
```

If the secondary shows a lower epoch than the primary, the secondary has not yet pulled the latest period — re-run `radosgw-admin period pull` followed by `radosgw-admin period update --commit`.

---

## 2. Sync Status

This is the single most important command in multisite operations. Run it on **both** sides.

```bash
# on primary-node and on secondary-node
radosgw-admin sync status
```

A healthy secondary looks roughly like this:

```
          realm <realm-id> (realm-a)
      zonegroup <zg-id> (zonegroup-a)
           zone <zone-b-id> (zone-b)
  metadata sync syncing
                full sync: 0/64 shards
                incremental sync: 64/64 shards
                metadata is caught up with master
      data sync source: <zone-a-id> (zone-a)
                        syncing
                        full sync: 0/128 shards
                        incremental sync: 128/128 shards
                        data is caught up with source
```

The two phrases you want to see are **`metadata is caught up with master`** and **`data is caught up with source`**.

On `primary-node` you'll see the inverse — `zone-a` reports `zone-b` as a sync target rather than a source.

---

## 3. Bucket Replication Test

Metadata replicates bidirectionally. Create a bucket on one side and confirm it appears on the other.

### 3.1 — Create a bucket on `primary-node`

You'll need the system user's keys (or any tenant user's keys) and an S3 client. The examples below use `s3cmd`; `aws-cli` works equivalently.

```bash
# on operator workstation, talking to primary endpoint
export AWS_ACCESS_KEY_ID=<USER_ACCESS_KEY>
export AWS_SECRET_ACCESS_KEY=<USER_SECRET_KEY>

aws --endpoint-url http://10.0.0.10:7474 s3 mb s3://replication-test-from-a
aws --endpoint-url http://10.0.0.10:7474 s3 ls
```

### 3.2 — Confirm the bucket appears on `secondary-node`

Wait 5–15 seconds (metadata sync is fast but not instant), then list buckets against the secondary endpoint:

```bash
aws --endpoint-url http://10.0.0.20:7474 s3 ls
```

You should see `replication-test-from-a` listed. If not, recheck `radosgw-admin sync status` on the secondary — it will pinpoint whether metadata sync is stalled.

### 3.3 — Reverse-direction smoke test

Repeat in reverse to confirm bidirectional metadata sync:

```bash
aws --endpoint-url http://10.0.0.20:7474 s3 mb s3://replication-test-from-b
# verify on primary:
aws --endpoint-url http://10.0.0.10:7474 s3 ls
```

---

## 4. Object Replication Test

### 4.1 — Upload an object on `primary-node`

```bash
# create a small test payload
dd if=/dev/urandom of=/tmp/payload.bin bs=1M count=4
sha256sum /tmp/payload.bin    # remember this hash

aws --endpoint-url http://10.0.0.10:7474 s3 cp /tmp/payload.bin \
    s3://replication-test-from-a/payload.bin
```

### 4.2 — Pull the object back from `secondary-node`

```bash
# allow a few seconds for data sync
sleep 10

aws --endpoint-url http://10.0.0.20:7474 s3 cp \
    s3://replication-test-from-a/payload.bin /tmp/payload-from-b.bin

sha256sum /tmp/payload-from-b.bin
```

The two SHA-256 hashes must match. If they do, **object data is replicating end-to-end across both sites**.

### 4.3 — Sanity check at the radosgw layer

For a Ceph-internal view, list objects directly through `radosgw-admin` on each side:

```bash
# on primary-node
radosgw-admin bucket stats --bucket=replication-test-from-a

# on secondary-node
radosgw-admin bucket stats --bucket=replication-test-from-a
```

`num_objects` and `size_actual` should match on both sides once sync settles.

---

## 5. Failure-Mode Diagnostics

When something doesn't replicate, these are the first three places to look — in this order.

### 5.1 — Sync error list

Both metadata and data sync workers log errors into per-shard queues:

```bash
# on secondary-node
radosgw-admin sync error list

# clear acknowledged errors once investigated
radosgw-admin sync error trim
```

Common causes that show up here: expired/wrong system-user keys, RGW endpoint unreachable, period mismatch.

### 5.2 — Per-source data sync status

For finer-grained diagnostics on the secondary, ask specifically about the source zone:

```bash
# on secondary-node
radosgw-admin data sync status --source-zone=zone-a
radosgw-admin metadata sync status
```

These reports list per-shard markers. If specific shards are stuck at the same marker over multiple invocations, that shard's sync has stalled.

### 5.3 — RGW pod logs

When `radosgw-admin` is uninformative, drop down to the pod logs:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-rgw --tail=200 -f
```

Search for `ERROR` and `WARN` lines mentioning `data sync`, `metadata sync`, or `realm`. Authentication issues against the master typically surface here as `403` responses from `10.0.0.10:7474`.

---

## ✅ Acceptance Criteria

The PoC is considered successful when **all** of the following hold:

- [ ] `radosgw-admin sync status` on both nodes reports `caught up` for both metadata and data.
- [ ] A bucket created via the `10.0.0.10` endpoint is visible via the `10.0.0.20` endpoint within ~30 seconds.
- [ ] A bucket created via the `10.0.0.20` endpoint is visible via the `10.0.0.10` endpoint within ~30 seconds.
- [ ] An object PUT to `10.0.0.10` can be GET'd from `10.0.0.20` with an identical SHA-256 within ~60 seconds.
- [ ] `radosgw-admin sync error list` is empty (or contains only previously-trimmed entries).