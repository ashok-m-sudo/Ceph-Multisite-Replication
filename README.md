# Ceph Multisite Replication — Two-Node Proof of Concept

> A two-server prototype demonstrating asynchronous, cross-site object storage replication using **Rook-Ceph** and the **RADOS Gateway (RGW)** multisite feature.

[![Rook](https://img.shields.io/badge/Rook-v1.16.5-EF5C55?style=flat-square&logo=rook&logoColor=white)](https://rook.io)
[![Ceph](https://img.shields.io/badge/Ceph-Object_Gateway-EF5C55?style=flat-square&logo=ceph&logoColor=white)](https://ceph.io)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)

---

## 📖 Project Overview

This repository documents a **two-node proof-of-concept** for Ceph multisite object storage replication. The setup uses Rook to deploy Ceph on Kubernetes at two independent sites and configures **asynchronous, S3-compatible replication** between them.

The goal is to validate, in a minimal lab environment, the core operational primitives required to run Ceph multisite in production:

- Bootstrapping a master zone on the primary site.
- Pulling realm and zonegroup metadata from a secondary site.
- Establishing bidirectional metadata sync and unidirectional data sync (master → secondary).
- Verifying that objects written to either side become visible across both.

This is **not** a production deployment guide — it is a portable lab recipe intended to make the topology explicit, the commands reproducible, and the failure modes observable.

---

## 🏗️ Architecture

### Topology

The prototype uses two independent Kubernetes clusters, one per site, each running its own Rook-Ceph stack. The RGW pods at each site are exposed via a Kubernetes `LoadBalancer` / external-IP service so the two sites can reach each other over HTTP on port `7474`.

```
                         ┌─────────────────────────────────┐
                         │       Realm: realm-a            │
                         │                                 │
                         │  ┌───────────────────────────┐  │
                         │  │   Zonegroup: zonegroup-a  │  │
                         │  │       (master group)      │  │
                         │  └───────────────────────────┘  │
                         └───────────────┬─────────────────┘
                                         │
                ┌────────────────────────┴────────────────────────┐
                │                                                 │
                ▼                                                 ▼
   ┌───────────────────────────┐                    ┌───────────────────────────┐
   │   Site A (primary-node)   │                    │  Site B (secondary-node)  │
   │   ─────────────────────   │  async replication │   ─────────────────────   │
   │   Zone: zone-a (master)   │ ◄────────────────► │   Zone: zone-b (replica)  │
   │   RGW: 10.0.0.10:7474     │                    │   RGW: 10.0.0.20:7474     │
   │   Rook-Ceph on K8s        │                    │   Rook-Ceph on K8s        │
   └───────────────────────────┘                    └───────────────────────────┘
```

### Ceph Multisite Concepts

| Resource | Name in this PoC | Role |
| :--- | :--- | :--- |
| **Realm** | `realm-a` | Top-level namespace containing all metadata for the global object store. There is exactly one realm in this PoC. |
| **Zonegroup** | `zonegroup-a` | Logical grouping of zones (think "region"). Designated as the **master zonegroup** — it owns metadata writes. |
| **Master Zone** | `zone-a` (Site A) | Source of truth. Accepts client writes for users, buckets, and objects, then pushes changes downstream. |
| **Secondary Zone** | `zone-b` (Site B) | Pulls realm/period metadata and replicates data asynchronously from `zone-a`. Can serve read traffic and accept writes that are then federated. |
| **Period** | (auto-generated) | An immutable snapshot of the realm configuration. Every change to zones/zonegroups must be committed via a new period. |
| **System User** | `synchronization-user` | A dedicated radosgw user with `--system` privileges whose access/secret keys are used by `zone-b` to authenticate to `zone-a`'s RGW. |

### Replication Model

- **Metadata sync**: bidirectional. Users, buckets, and ACLs created on either side propagate both ways.
- **Data sync**: object payloads replicate asynchronously from the master zone to the secondary. Convergence is eventual, typically within seconds in a healthy lab.

---

## ✅ Prerequisites

Before following the deployment guide, both sites must satisfy the following:

**On each node (Site A and Site B):**
- A working Kubernetes cluster (single-node or HA — this PoC was validated on single-node clusters).
- A healthy **Rook-Ceph cluster** already deployed in the `rook-ceph` namespace, with at least one OSD per node and `ceph -s` reporting `HEALTH_OK`.
- `kubectl` access with permission to create resources in the `rook-ceph` namespace.
- Outbound and inbound network reachability to the peer site's RGW endpoint on **TCP/7474**.

**Tooling:**
- `git`, `kubectl`, `telnet` (or `nc`) on the operator workstation.
- The Rook examples checked out at the matching version:
  ```bash
  git clone --single-branch --branch v1.16.5 https://github.com/rook/rook.git
  cd rook/deploy/examples
  ```

**Network plan (used throughout this repo):**

| Site | Hostname | RGW External IP | Port |
| :--- | :--- | :--- | :--- |
| Site A | `primary-node` | `10.0.0.10` | `7474` |
| Site B | `secondary-node` | `10.0.0.20` | `7474` |

> ⚠️ Replace these placeholders with your own routable IPs before applying any manifest. They are used as conspicuous markers throughout the docs so they're easy to find with grep.

---

## ⚡ Quickstart Summary

The full procedure is in [`docs/deployment-guide.md`](docs/deployment-guide.md). At a glance:

1. **On `primary-node` (Site A):** apply `object-multisite.yaml` to create `realm-a`, `zonegroup-a`, `zone-a`, and the object store.
2. **On `primary-node`:** expose RGW on `10.0.0.10:7474` via `rgw-external.yaml`.
3. **On `primary-node`:** create the `synchronization-user` system user, attach its keys to `zone-a`, and commit a new period.
4. **On `secondary-node` (Site B):** apply `object-multisite-pull-realm.yaml` referencing `http://10.0.0.10:7474` and the system user credentials — this pulls `realm-a` and creates `zone-b`.
5. **On `secondary-node`:** expose RGW on `10.0.0.20:7474`, register the endpoint against `zone-b`, and commit a new period.
6. **Verify** with `radosgw-admin sync status` on both sides — see [`docs/verification.md`](docs/verification.md).

---

## 📂 Repository Structure

```
ceph-multisite-poc/
├── README.md                                   ← you are here
├── docs/
│   ├── deployment-guide.md                     ← step-by-step bring-up
│   └── verification.md                         ← end-to-end replication tests
├── configs/
│   ├── primary/                                ← manifests for Site A (zone-a, master)
│   │   ├── object-multisite.yaml
│   │   └── rgw-external.yaml
│   └── secondary/                              ← manifests for Site B (zone-b, replica)
│       ├── object-multisite-pull-realm.yaml
│       └── rgw-external.yaml
└── .gitignore
```

---

## 📚 References

- [Rook documentation — Object Multisite](https://rook.io/docs/rook/latest/CRDs/Object-Storage/ceph-object-multisite-crd/)
- [Ceph documentation — Multisite Configuration](https://docs.ceph.com/en/latest/radosgw/multisite/)
- [Rook examples at v1.16.5](https://github.com/rook/rook/tree/v1.16.5/deploy/examples)

---

## 📝 License

MIT — see [LICENSE](LICENSE). This repository contains documentation only; it is provided as-is, without warranty.