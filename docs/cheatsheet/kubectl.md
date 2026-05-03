# kubectl

## Find broken things first

```bash
# Pods that are not Running:
kubectl get pod -A --field-selector=status.phase!=Running

# Pods where not all containers are Ready (ready != desired):
kubectl get pod -A -o json \
  | jq -r '.items[] | select(.status.containerStatuses != null)
           | select([.status.containerStatuses[].ready] | all | not)
           | "\(.metadata.namespace)/\(.metadata.name)  \(.status.phase)"'

# Recent warnings cluster-wide, newest last:
kubectl get events -A --sort-by=.lastTimestamp \
  | grep -v Normal | tail -40

# All LoadBalancer Services and their assigned IPs:
kubectl get svc -A --field-selector spec.type=LoadBalancer \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,EXTIP:.status.loadBalancer.ingress[0].ip
```

## Logs (the most-used commands in any incident)

```bash
# Live tail
kubectl logs -n <ns> <pod> -c <container> -f

# Previous container, after a crash. This is where the actual error usually is.
kubectl logs -n <ns> <pod> -c <container> --previous --tail=200

# All containers in a pod, prefixed:
kubectl logs -n <ns> <pod> --all-containers --prefix --tail=200

# Filter the chatty health-probe noise:
kubectl logs -n <ns> <pod> | grep -vE "GET /(healthz|readyz|metrics)"
```

## Watch readiness without polling by hand

`kubectl wait` is purpose-built. Use it inside scripts.

```bash
kubectl wait pod -n <ns> -l <selector> --for=condition=Ready --timeout=120s

# Or if you want the pod-level Phase:
kubectl wait pod -n <ns> -l <selector> --for=jsonpath='{.status.phase}'=Running --timeout=120s
```

For "until both of these are Ready," loop on jsonpath:

```bash
until \
  [ "$(kubectl get pod -n monitoring  -l app.kubernetes.io/name=grafana \
        -o jsonpath='{.items[0].status.containerStatuses[?(@.name=="grafana")].ready}')" = "true" ] \
  && \
  [ "$(kubectl get pod -n automation -l app=claude-bridge \
        -o jsonpath='{.items[0].status.containerStatuses[0].ready}')" = "true" ]; do
  sleep 4
done
echo READY
```

## Inspect a resource quickly

```bash
# YAML for any resource (works for CRDs):
kubectl get <kind> -n <ns> <name> -o yaml

# Just the status:
kubectl get pod -n <ns> <name> -o jsonpath='{.status.conditions}' | jq .

# Just the container ready states:
kubectl get pod -n <ns> <name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{": ready="}{.ready}{" restarts="}{.restartCount}{"\n"}{end}'

# Custom columns for a list view:
kubectl get pod -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
```

## Restart things

```bash
# Roll a Deployment / DaemonSet / StatefulSet without changing the spec:
kubectl rollout restart deployment/<name> -n <ns>

# Skip CrashLoopBackOff backoff for a single pod:
kubectl delete pod -n <ns> <pod>           # the controller recreates it immediately

# Drain a node before maintenance:
kubectl cordon  <node>
kubectl drain   <node> --ignore-daemonsets --delete-emptydir-data --grace-period=60
# ... do work ...
kubectl uncordon <node>
```

## CRDs and "I forgot the API group"

```bash
# What kinds exist?
kubectl api-resources | grep -i <substring>

# Schema for a kind, deeply:
kubectl explain ipaddresspool.spec --api-version=metallb.io/v1beta1 --recursive
```

## kubeconfig and contexts

```bash
kubectl config get-contexts
kubectl config use-context <name>
kubectl config view --minify --raw          # current context only, with secrets
```

## etcd operations on a kubeadm cluster

Run from a control-plane node. The cert paths assume the kubeadm default; adjust if your setup is different.

```bash
ETCD_ENDPOINTS=https://192.168.1.89:2379,https://192.168.1.90:2379,https://192.168.1.91:2379
ETCD_CERTS="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
            --cert=/etc/kubernetes/pki/etcd/server.crt \
            --key=/etc/kubernetes/pki/etcd/server.key"

# What's the on-disk size of each member's DB?
sudo ETCDCTL_API=3 etcdctl --endpoints=$ETCD_ENDPOINTS $ETCD_CERTS \
  endpoint status -w table

# Compaction: discards historical revisions. Logical, cluster-wide, fast.
REV=$(sudo ETCDCTL_API=3 etcdctl --endpoints=$ETCD_ENDPOINTS $ETCD_CERTS \
        endpoint status -w json | jq '.[0].Status.header.revision')
sudo ETCDCTL_API=3 etcdctl --endpoints=$ETCD_ENDPOINTS $ETCD_CERTS compact $REV

# Defragmentation: rewrites the .db file to reclaim disk. Per-member.
# Run one at a time, wait for endpoint health between members.
for ep in https://192.168.1.89:2379 https://192.168.1.90:2379 https://192.168.1.91:2379; do
  sudo ETCDCTL_API=3 etcdctl --endpoints=$ep $ETCD_CERTS defrag
  sudo ETCDCTL_API=3 etcdctl --endpoints=$ep $ETCD_CERTS endpoint health
done

# Alarms (raised on NOSPACE, etc.)
sudo ETCDCTL_API=3 etcdctl --endpoints=$ETCD_ENDPOINTS $ETCD_CERTS alarm list
sudo ETCDCTL_API=3 etcdctl --endpoints=$ETCD_ENDPOINTS $ETCD_CERTS alarm disarm
```

Compaction and defragmentation are different operations with confusingly similar names. Compaction reduces logical size; defragmentation reclaims disk. You need both. See the [etcd-defrag runbook](../runbooks/etcd-defrag.md).

## ExternalSecrets / secrets management

```bash
# Force a single ExternalSecret to re-sync immediately:
kubectl annotate externalsecret <name> -n <ns> \
  force-sync=$(date +%s) --overwrite

# Show last sync status across all ExternalSecrets:
kubectl get externalsecret -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,STATUS:.status.conditions[0].reason,SYNCED:.status.conditions[0].status,LASTSYNC:.status.refreshTime

# Scale ESO down (stops the retry loop during a rate-limit incident):
kubectl -n external-secrets scale deploy external-secrets               --replicas=0
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=0
kubectl -n external-secrets scale deploy external-secrets-webhook       --replicas=0

# Bring it back:
kubectl -n external-secrets scale deploy external-secrets               --replicas=1
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=1
kubectl -n external-secrets scale deploy external-secrets-webhook       --replicas=1

# What is this Secret actually storing? (decoded values)
kubectl get secret -n <ns> <name> -o json \
  | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

## Find pods by restart count

A leading indicator of a crashloop or a flaky dependency. Often the first signal of underlying control-plane instability (etcd bloat, scheduler problems).

```bash
# Top 20 pods cluster-wide by restart count of the first container:
kubectl get pod -A --sort-by=.status.containerStatuses[0].restartCount \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,STATUS:.status.phase \
  | tail -20

# Anything that has restarted more than 5 times in any container:
kubectl get pod -A -o json \
  | jq -r '.items[] | select(.status.containerStatuses != null)
           | select([.status.containerStatuses[].restartCount] | max > 5)
           | "\(.metadata.namespace)/\(.metadata.name) restarts=\([.status.containerStatuses[].restartCount] | max)"'
```

## Things I reach for during an incident

Grouped by question.

| Question | Command |
|----------|---------|
| Why is this pod not Ready? | `kubectl describe pod -n <ns> <pod> \| tail -40` |
| What did the previous container say before crashing? | `kubectl logs -n <ns> <pod> -c <c> --previous --tail=200` |
| Has anything restarted lately? | `kubectl get pod -A --sort-by=.status.containerStatuses[0].restartCount \| tail` |
| Does this MetalLB IP have a Ready endpoint? | `kubectl get endpointslices -A \| grep <svc>` |
| What does the speaker think about this IP? | `kubectl logs -n metallb-system -l component=speaker --tail=400 \| grep -E "<svc>\|<ip>\|notOwner"` |
| Did something just rollout? | `kubectl get events -A --sort-by=.lastTimestamp \| tail -40` |
| Is this resource ArgoCD-managed? | `kubectl get <kind> <name> -n <ns> -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}'` |

## Patterns worth knowing

- **Always read `--previous` logs after a crash.** The live container is in backoff and has nothing to say.
- **Pod conditions over `phase`.** `phase=Running` does not mean the pod is healthy; check `Ready` and `ContainersReady`.
- **`kubectl wait` is better than sleep.** Avoids race conditions and times out cleanly.
- **`-o jsonpath` is faster than piping `-o yaml` into grep.** It also does not break when YAML formatting changes.
