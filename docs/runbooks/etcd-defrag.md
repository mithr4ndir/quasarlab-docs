# Runbook: etcd database bloat and defragmentation

## Symptom

One or more of:

- `kube-controller-manager` and `kube-scheduler` pods crash-looping with high restart counts (12, 16, 20+ over a few hours).
- API server returning intermittent `request timed out` and `leader changed` errors on writes.
- `etcd` peer logs full of "took too long" warnings and slow apply messages.
- ArgoCD `OutOfSync` on healthy apps because reconciliation is timing out.
- Lease renewal failures across multiple controller-managers.

The pattern is leader-election instability driven by GC pressure inside etcd itself. The cluster keeps "kind of" working because reads are fine, but anything that needs an etcd write is gambling.

## Triage

```bash
# Look for the "request timed out" pattern in apiserver logs
kubectl logs -n kube-system -l component=kube-apiserver --tail=200 \
  | grep -iE "etcd|leader|timed out"

# Controller-manager restart counts
kubectl -n kube-system get pod -l component=kube-controller-manager \
  -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount

# Direct etcd health (run from one of the control-plane nodes, as root):
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table

# Look at the "DB SIZE" column. Anything over ~100 MB is the warning zone.
# Anything over ~200 MB is "fix this now."
```

The `DB SIZE` column is the on-disk file size of `member.db`. It does **not** equal the working-set of keys stored. etcd never shrinks the file on its own; deleted revisions leave fragmentation that only `etcdctl defrag` reclaims.

## Why this happens

etcd has two distinct cleanup operations and the names sound similar:

| Operation | What it does | What it does NOT do |
|-----------|--------------|---------------------|
| **Compaction** (`compact`) | Discards historical revisions older than a watermark. Reduces logical size. | Does NOT reclaim disk space. The freed space stays in the file as fragmentation. |
| **Defragmentation** (`defrag`) | Rewrites the `member.db` file to be tightly packed. Reclaims disk. | Does NOT discard any data on its own. |

You need **both**, and you need them periodically. kubeadm's default etcd config has `auto-compaction-retention` *unset*, so by default neither happens. A cluster that has been up for a year without intervention is the textbook case for this incident.

## Fix

This is a per-member operation. etcd in a HA cluster has 3 members and you defrag them one at a time, waiting for each to come back to `Healthy` before moving on. Defrag blocks writes on the member being processed, but the other members keep serving.

### Step 1: compact

Compact retains everything from a recent revision forward. Pick a revision to keep up to; `endpoint status` returns the current revision.

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json | jq '.[0].Status.header.revision'

REV=<that-number>

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact $REV
```

Compaction is fast (seconds) and is a logical cluster-wide operation; you only run it once.

### Step 2: defrag, member by member

```bash
# k8cluster1
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://192.168.1.90:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag

# wait for endpoint health to return Healthy, then move on
sudo ETCDCTL_API=3 etcdctl ... endpoint health

# k8cluster2 -> k8cluster3, same pattern
```

After all three: `endpoint status` should show `DB SIZE` dropped substantially (a 124 MB → 46 MB drop is realistic). Within a minute or two, controller-manager restart counts stop climbing and the apiserver `request timed out` rate falls to zero.

### Step 3: lift the alarm if any

If etcd raised a NOSPACE alarm (it can if disk pressure was severe), clear it:

```bash
sudo ETCDCTL_API=3 etcdctl ... alarm list
sudo ETCDCTL_API=3 etcdctl ... alarm disarm
```

## Prevention

The lab runs a weekly defrag timer via the `k8s_maintenance` Ansible role:

- `Sunday 04:00 UTC`, a systemd timer on each control-plane node runs `kubectl exec` against the local etcd member, performs compact + defrag, and writes Prometheus textfile metrics for last-run time and resulting DB size.
- Two PrometheusRules:
  - `EtcdDbSizeLarge` — warning at 100 MB sustained for 1 hour.
  - `EtcdDbSizeCritical` — page at 200 MB sustained for 15 minutes.

There is one known gap: kube-prometheus-stack disables the `kubeEtcd` ServiceMonitor by default because etcd binds to `127.0.0.1` on kubeadm clusters. The alerts above reference `etcd_mvcc_db_total_size_in_bytes`, which is exposed by etcd itself but not currently scraped. Tracked as a follow-up; for now, the textfile metrics from the maintenance role are what the alerts read instead.

## Patterns worth knowing

- **Compaction is logical, defrag is physical.** They are separate operations and you need both.
- **Defrag one member at a time, wait for healthy.** Doing them in parallel can lose quorum.
- **The DB size column is the only durable signal.** Counting keys, looking at GC stats, watching apiserver latency are all proxies for what `endpoint status -w table` will show you in one line.
- **`auto-compaction-retention` is off by default on kubeadm.** Either turn it on cluster-wide or run a periodic job. The lab does the latter so the timing is observable.
