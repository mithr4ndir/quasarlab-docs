# Helm and ArgoCD

## Helm: what is installed and what values are in use

```bash
helm list -A
helm get values <release> -n <ns>           # values currently applied
helm get manifest <release> -n <ns>         # full rendered manifest
helm get all <release> -n <ns>              # values + manifest + hooks
helm history <release> -n <ns>
helm diff upgrade <release> <chart> -f values.yaml -n <ns>   # plugin: helm-diff
```

## Helm: install or upgrade

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install <release> <chart> \
  -n <ns> --create-namespace \
  -f values.yaml \
  --version <pinned>      # always pin in this lab
```

## ArgoCD CLI: day-to-day

```bash
argocd login argocd.herro.me --sso          # or --username admin
argocd app list
argocd app get <app>
argocd app diff <app>                       # spec drift before sync
argocd app sync <app>
argocd app sync <app> --resource <kind>:<name>
argocd app history <app>
argocd app rollback <app> <history-id>
```

## ArgoCD: when something is "OutOfSync" but I do not want to sync yet

```bash
# Pause auto-sync briefly:
argocd app set <app> --sync-policy none

# Force a refresh from Git without applying:
argocd app get <app> --refresh

# Hard refresh (re-reads from Git, ignoring any caches):
argocd app get <app> --hard-refresh
```

## App-of-Apps

The `root` Application points at `argocd-apps/` in `k8s-argocd`. Every other Application is a file in that directory. New apps are one PR. Sync waves order them via `argocd.argoproj.io/sync-wave`.

## Things I reach for during an incident

| Question | Command |
|----------|---------|
| Is this resource being managed by ArgoCD at all? | `kubectl get <kind> <name> -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'` |
| Will my live edit be reverted? | If `tracking-id` exists, yes. Pause the app first or change Git. |
| What is currently deployed for this release? | `helm get values <release> -n <ns>` |
| What changed in the last sync? | `argocd app history <app>` and `argocd app diff <app>` |
