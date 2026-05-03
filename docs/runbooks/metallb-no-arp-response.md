# Runbook: MetalLB IP not answering ARP

## Symptom

A LoadBalancer IP in the `192.168.1.225-240` pool stops responding from the LAN. `ping` times out, `curl` cannot connect, but the underlying app has at least one Running pod and the Service still shows the EXTERNAL-IP.

## Triage

Three things to check, in order:

```bash
# 1. Is there a Ready endpoint? "No ready endpoints" is the most common cause.
kubectl get endpointslices -A | grep <service-name>
kubectl get pod -n <ns> -l <selector> -o wide

# 2. What do the speakers say? Look for "notOwner" or "serviceWithdrawn".
for p in $(kubectl -n metallb-system get pod -l component=speaker -o name); do
  echo "=== $p ==="
  kubectl -n metallb-system logs --tail=400 $p \
    | grep -E "<service-name>|<lb-ip>|notOwner|Withdrawn|Announced"
done

# 3. ARP table from a host on the same L2 segment:
ip neigh | grep 192.168.1.<lb-ip>
arping -c 3 192.168.1.<lb-ip>   # if installed
```

Decision tree:

| Finding | Cause | Fix |
|---------|-------|-----|
| `serviceWithdrawn ... reason=notOwner` and no `serviceAnnounced` from any speaker | No node has a Ready endpoint | Fix readiness on the underlying pod |
| `serviceAnnounced` from a speaker but no ARP reply | Network policy or firewall on the node, or the elected speaker pod is unhealthy | Check speaker pod, host firewall, Calico policies |
| EXTERNAL-IP missing | Pool exhausted or controller down | `kubectl get pod -n metallb-system` and `kubectl describe svc` |

## Fix (the "no ready endpoints" case)

MetalLB itself does not need to be touched. Fix readiness on the underlying app and MetalLB will re-elect a speaker and re-announce the IP automatically.

```bash
kubectl describe pod -n <ns> <pod>          # readiness probe details, recent events
kubectl logs -n <ns> <pod> --previous       # if it has crashed
```

If the readiness regression turns out to be a backend-auth issue, see the [pg_hba runbook](pg-hba-rejecting-k8s-pods.md).

## Why this happens

MetalLB in L2 mode will only advertise an IP from a node that has at least one Ready endpoint cluster-wide. When the last endpoint flips to NotReady, the elected speaker logs `serviceWithdrawn ... reason=notOwner` and stops answering ARP for that IP. This is correct behavior: there is nothing healthy to send traffic to. Once any endpoint becomes Ready again, a speaker takes ownership and ARP starts working.

## Quick check that MetalLB itself is fine

```bash
kubectl get ipaddresspool,l2advertisement -n metallb-system
kubectl get pod -n metallb-system -o wide      # 1 controller + N speakers, all Running
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

If all three look right and the right speaker is logging `serviceAnnounced` for your IP, the problem is downstream of MetalLB.
