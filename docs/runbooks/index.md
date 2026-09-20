# Runbooks

Short, fix-it-now docs derived from real incidents in the lab. Each runbook follows the same shape:

1. **Symptom you would actually see.**
2. **Quick triage** to confirm.
3. **Fix.**
4. **Follow-up** so it does not bite again.

## Kubernetes platform

- [Postgres rejecting K8s pods (`pg_hba.conf`)](pg-hba-rejecting-k8s-pods.md)
- [MetalLB IP not answering ARP](metallb-no-arp-response.md)
- [Grafana CrashLoopBackOff](grafana-crashloop.md)
- [etcd database bloat and defragmentation](etcd-defrag.md)
- [Bitnami image 404 / `ImagePullBackOff`](bitnami-image-404.md)

## Storage

- [Reboot the NAS with zero VM downtime](nas-reboot-evacuate-vm-disks.md)
- [Migrate to tiered NAS storage](nas-storage-tiering-migration.md)

## Apps

- [Jellyfin database corruption from ungraceful shutdown](jellyfin-db-corruption.md)
- [Jellyfin transcoding and ffmpeg](jellyfin-transcoding-ffmpeg.md)
- [Jellyfin transcode cache filling the root filesystem](jellyfin-transcode-cache-disk-full.md)
- [GPU passthrough D3cold recovery (jellyfin / pve2 / RTX 2080 Ti)](gpu-passthrough-d3cold-recovery.md)

## Secrets and quotas

- [1Password rate-limit hit](1password-rate-limit.md)
