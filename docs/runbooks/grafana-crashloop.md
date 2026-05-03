# Runbook: Grafana CrashLoopBackOff

## Symptom

`kube-prometheus-stack-grafana-*` pod shows `2/3 Running, CrashLoopBackOff`. The two healthy containers are the `grafana-sc-dashboard` and `grafana-sc-datasources` k8s-sidecar containers. The crashing one is `grafana` itself.

## Triage

```bash
POD=$(kubectl -n monitoring get pod -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}')

# Why is the grafana container exiting?
kubectl -n monitoring logs $POD -c grafana --previous --tail=200

# Pod-level events:
kubectl -n monitoring describe pod $POD | tail -40
```

Read the **previous** container logs, because the live container is in backoff and has no logs of its own.

Common patterns:

| Log signature | Likely cause | See |
|---------------|--------------|-----|
| `pq: no pg_hba.conf entry for host ...` | Postgres rejecting the new source IP | [pg_hba runbook](pg-hba-rejecting-k8s-pods.md) |
| `dial tcp <postgres-ip>:5432: i/o timeout` | Network reachability, firewall, or Postgres down | check `nc -vz <ip> 5432`, `ssh` and `systemctl status postgresql@16-main` |
| `Error parsing config file ... grafana.ini` | ConfigMap rolled out by Reloader has a typo | `kubectl -n monitoring describe cm kube-prometheus-stack-grafana` and revert in Git |
| `failed to load plugin` | `GF_INSTALL_PLUGINS` references a plugin that is no longer compatible with the Grafana image version | pin the plugin or upgrade Grafana |

## Fix

Once the underlying cause is fixed, kick the pod to skip the backoff window:

```bash
kubectl delete pod -n monitoring \
  -l app.kubernetes.io/name=grafana
kubectl wait pod -n monitoring \
  -l app.kubernetes.io/name=grafana --for=condition=Ready --timeout=120s
```

## Note on the ConfigMap being ArgoCD-managed

`kube-prometheus-stack-grafana` is reconciled by ArgoCD. Editing the ConfigMap in-cluster will be reverted on next sync. If the fix is a config change, do it in the Helm values in Git and let ArgoCD apply it. Direct `kubectl edit` is fine only as a temporary mitigation, and you should pause the ArgoCD app before doing so:

```bash
argocd app set kube-prometheus-stack --sync-policy none
# ... edit ...
# put the same change in Git, then:
argocd app set kube-prometheus-stack --sync-policy automated
argocd app sync kube-prometheus-stack
```

## Reloader interaction

The pod has `reloader.stakater.com/auto: "true"`. Any change to the ConfigMap or the linked Secret (`grafana-secrets`, `kube-prometheus-stack-grafana`) will trigger a rollout. If Grafana started crashing right after a quiet ConfigMap or Secret change, that is your trigger window.
