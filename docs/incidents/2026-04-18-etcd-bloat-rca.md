# 2026-04-18: etcd bloat, control-plane instability, the missing weekly defrag

**Date:** symptoms 2026-04-16 to 2026-04-18, RCA finished 2026-04-18
**Severity:** S2. Control plane unhealthy, no data loss, workloads kept serving but reconciliation got slow and noisy.

## Symptom

`kube-controller-manager` had restarted 12, 16, and 8 times across the three control-plane nodes over the prior 36 hours. ArgoCD was showing apps drift in and out of `OutOfSync` for no apparent reason. The NFS provisioner had restarted 16 times in 11 days. The apiserver log was full of:

```text
W ... etcdserver: read-only range request "..." took too long (xxxx ms) ...
W ... etcdserver: leader changed
E ... timed out waiting for the condition
```

Plus controller-managers logging "leader lease expired" and re-electing every few minutes. None of this was causing a hard outage; it was the "100 papercuts" version of an unhealthy cluster.

## Investigation

Direct etcd health was the first thing to check, and it told the whole story in one line:

```text
$ etcdctl ... endpoint status -w table
+----------------------------+------------------+---------+---------+-----------+...+----------+
|         ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER |...|          |
+----------------------------+------------------+---------+---------+-----------+...+----------+
| https://192.168.1.89:2379  | a1b2c3d4...      | 3.5.x   |  124 MB | false     |...|          |
| https://192.168.1.90:2379  | b2c3d4e5...      | 3.5.x   |  128 MB | true      |...|          |
| https://192.168.1.91:2379  | c3d4e5f6...      | 3.5.x   |  126 MB | false     |...|          |
+----------------------------+------------------+---------+---------+-----------+...+----------+
```

The `member.db` file on each node was around 125 MB. The cluster had been up 267 days with no defrag ever run. Every revision deleted by Kubernetes garbage collection had freed logical space inside the etcd database but not physical space on disk, so the file kept growing. Eventually, GC pauses inside boltdb (etcd's storage engine) started exceeding the etcd heartbeat interval, leader elections flapped, and lease renewals timed out.

## Five whys

1. controller-manager crash-looped because its leader lease renewal failed (5 second timeout)
2. lease renewal failed because the apiserver could not complete the PUT to etcd within timeout
3. apiserver was slow because etcd returned `request timed out` and `leader changed`
4. etcd was unstable because a 124 MB on-disk DB caused GC pauses longer than the heartbeat
5. the DB had grown unchecked for 267 days because no automated defrag was configured

## Root cause

kubeadm's default etcd configuration does not set `auto-compaction-retention`, so etcd never discards old revisions on its own. Kubernetes itself does delete-and-recreate workflows constantly (every Pod status update, every lease renewal), and each deletion leaves a tombstone that compaction would clean up. Without compaction, the DB grows monotonically. And even if compaction *had* been enabled, **it would not have shrunk the on-disk file** because that is what defragmentation does, separately.

Two distinct operations, named confusingly:

- **Compaction**: discards historical revisions older than a watermark. Reduces logical size. Does not touch the file.
- **Defragmentation**: rewrites `member.db` to be tightly packed. Reclaims disk. Does not discard data.

You need both, periodically. I had neither.

## Fix

```bash
# On any control-plane node, get the current revision:
sudo ETCDCTL_API=3 etcdctl ... endpoint status --write-out=json \
  | jq '.[0].Status.header.revision'

# Compact, cluster-wide, to that revision:
sudo ETCDCTL_API=3 etcdctl ... compact 48258164

# Defrag each member, one at a time, waiting for endpoint health between:
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.90:2379 ... defrag
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.89:2379 ... defrag
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.91:2379 ... defrag
```

`DB SIZE` dropped from 124 to 46 MB on the leader, similar drops on the followers. Within minutes:

- `etcdserver: request timed out` rate fell to zero.
- controller-manager restarts stopped.
- ArgoCD apps stabilized to `Synced`.
- NFS provisioner stopped restarting.

Same day, I rolled a Kubernetes upgrade that had been queued: all three nodes from `v1.33.10` to `v1.33.11`, kernel `6.8.0-110-generic`, no NVIDIA / GPU node hiccups beyond a known dpkg warning on `nvidia-container-toolkit` on `k8cluster2` that was unrelated.

## Preventive measures

- **Weekly etcd defrag timer** via the Ansible `k8s_maintenance` role (PR #107). Runs every Sunday at 04:00 UTC on each control-plane node, performs compact + defrag via `kubectl exec`, writes Prometheus textfile metrics for last-run timestamp and resulting DB size.
- **PrometheusRules** for early warning (PR #132):
  - `EtcdDbSizeLarge` warning at 100 MB sustained for 1 hour
  - `EtcdDbSizeCritical` page at 200 MB sustained for 15 minutes
- **Runbook** at [Runbooks / etcd defrag](../runbooks/etcd-defrag.md) so the next person (which is usually future me) does not have to rediscover the operation.

There is one open gap. kube-prometheus-stack ships with `kubeEtcd` ServiceMonitor disabled by default, because etcd binds to `127.0.0.1` on kubeadm clusters and is not directly scrapable from inside the cluster. The alerts above currently read from the textfile metrics written by the maintenance role. Exposing etcd metrics properly (cert-secured peer endpoint scraped by Prometheus) is a small piece of follow-up work tracked in the maintenance role's TODOs.

## What this incident is a good example of

- **A "who has been on call for 267 days?" failure mode.** Most monitoring systems will tell you the cluster is degraded *now*; very few will tell you the cluster is silently growing toward a slow-motion crash three months from now. The fix is operational hygiene with metrics, not a smarter alert.
- **Naming-induced confusion as a bug.** "Compaction" and "defragmentation" sound interchangeable and are not. A whole class of "I thought I had this configured" mistakes traces back to operators conflating them.
- **Symptom-of-symptom debugging.** The first guess (controller-manager leader election problems) was a real symptom but four whys away from the cause. Following the chain past the obvious one is the work.
