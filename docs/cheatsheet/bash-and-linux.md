# Bash and Linux

The shell tricks I use during incidents and day-to-day. Grouped by what I want to find out.

## Is a TCP port reachable, with no extra tools?

Bash has `/dev/tcp`. No `nc`, no `telnet`, no Docker required.

```bash
timeout 3 bash -c 'cat < /dev/tcp/192.168.1.123/5432' ; echo "exit=$?"
# exit=0  -> connected
# exit=1  -> connection refused / firewall
# exit=124 -> timeout (probably filtered)
```

For a quick set of ports across hosts:

```bash
for h in 192.168.1.{89,90,91,123}; do
  for p in 22 5432 6443; do
    timeout 1 bash -c "cat </dev/tcp/$h/$p" >/dev/null 2>&1 \
      && echo "$h:$p open" || echo "$h:$p closed"
  done
done
```

## What is listening on this host?

```bash
sudo ss -ltnp                       # listening TCP, with PID and process name
sudo ss -lunp                       # listening UDP
sudo lsof -i :3001                  # who has port 3001?

# Outbound established connections:
sudo ss -tnp state established
```

## Wait for something to be true (poll loops done right)

```bash
# Bash idiom for "until condition X, sleep, retry" with a timeout:
end=$(( $(date +%s) + 120 ))
until <check> ; do
  [ $(date +%s) -ge $end ] && { echo "timeout"; exit 1; }
  sleep 4
done
```

For Kubernetes specifically, prefer `kubectl wait`. For external resources, the loop above is the simplest portable version.

## SSH non-interactively (useful in scripts)

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new \
    user@host '<remote-command>'
```

`BatchMode=yes` makes SSH fail fast instead of prompting for a password. `ConnectTimeout=5` keeps a dead host from stalling a script. `accept-new` adds new hosts to `known_hosts` automatically but rejects host-key changes.

## Edit configs safely on a remote host

The pattern I use for any `pg_hba.conf`-class file: timestamped backup, append, reload.

```bash
ssh <host> "
  TS=\$(date +%Y%m%d-%H%M%S)
  sudo cp -a /etc/<app>/<file>.conf /etc/<app>/<file>.conf.bak-\${TS}
  sudo tee -a /etc/<app>/<file>.conf > /dev/null <<EOF

# my change
foo = bar
EOF
  sudo systemctl reload <unit>
"
```

`tee -a` instead of `>>` because `>>` runs locally and never reaches the remote shell when wrapped in `sudo`.

## What changed on this host today?

```bash
# Recently modified config files:
sudo find /etc -mtime -1 -type f 2>/dev/null

# Package install or upgrade activity:
grep -E "(install |upgrade )" /var/log/dpkg.log /var/log/dpkg.log.1 2>/dev/null | tail

# Service restarts in the last 6 hours:
journalctl --since "6 hours ago" \
  | grep -E "Stopped|Started|failed|reload"

# Last login times:
last -n 10
```

## journalctl, the parts you actually use

```bash
journalctl -u <unit> -f                  # live tail
journalctl -u <unit> --since "1 hour ago"
journalctl -u <unit> -p err              # priority err and worse
journalctl -u <unit> -k                  # kernel messages too
journalctl -u <unit> -o json | jq .      # structured
```

## Filter a chatty log to the parts you care about

```bash
# Drop all health-probe lines:
grep -vE "GET /(healthz|readyz|metrics)" run.log

# Show error-class lines only:
grep -E "ERROR|FATAL|panic|Traceback|failed|denied" run.log

# Group by minute:
awk '{print substr($0,1,16)}' run.log | sort | uniq -c | tail
```

## jq snippets I keep reaching for

```bash
# Compact array of items where a flag is false:
jq '[.items[] | select(.ready==false) | .name]'

# Filter by nested key with a default:
jq '.items[] | {name: .name, ip: (.status.podIP // "n/a")}'

# Group by namespace, count:
jq -r '.items | group_by(.metadata.namespace) | map({ns:.[0].metadata.namespace, n:length}) | .[] | "\(.ns)\t\(.n)"'
```

## ARP and neighbor table (for MetalLB debugging)

```bash
ip neigh                                 # current ARP/NDP table
ip neigh flush all                       # clear it (forces re-learn)
arping -c 3 192.168.1.229                # poke an IP, see who replies
```

## Process-tree triage (who is the runaway caller?)

When a rate-limited or quota-bound resource is being hammered and you do not know which script is doing it, walk the process tree from a parent that you recognize.

```bash
# Forest view, with PIDs and full command lines:
ps -eo pid,ppid,user,cmd --forest

# Filter to a likely culprit, and walk up the tree:
ps -eo pid,ppid,user,cmd --forest | grep -E "(ansible|op |run-)" | grep -v grep

# Show ancestors of a specific PID:
ps -o pid,ppid,user,cmd --forest -g $(ps -o pgid= -p <pid>)
```

This is how the dynamic-inventory `op read` storm was identified during the [04-19 cache-bypass incident](../incidents/2026-05-02-op-ratelimit-recurrence.md).

## systemd drop-ins (overrides without touching upstream units)

Drop-in directories are how you patch a vendor-provided systemd unit safely. The pattern shows up twice in the lab: the TrueNAS-API ExecStartPre on `pve-guests.service`, and the `op-quota-collector.timer` adjustments.

```bash
# 1. Create the directory:
sudo mkdir -p /etc/systemd/system/<unit>.service.d/

# 2. Write the override file (any name, must end in .conf):
sudo tee /etc/systemd/system/<unit>.service.d/local.conf > /dev/null <<'EOF'
[Service]
ExecStartPre=/usr/local/sbin/wait-for-something.sh
TimeoutStartSec=infinity
EOF

# 3. Reload + restart:
sudo systemctl daemon-reload
sudo systemctl restart <unit>

# 4. Verify the override is in effect:
sudo systemctl cat <unit>
systemd-analyze verify <unit>.service
```

`systemctl cat` shows the merged unit file (upstream + drop-ins) so you can confirm your override actually loaded.

## Probe a container registry without pulling

Useful when you suspect a vendor pulled the rug (e.g. the [Bitnami removal](../incidents/2026-04-19-bitnami-images-removed.md)) or you want to confirm a tag exists before referencing it.

```bash
# crane (from go-containerregistry) is the cleanest tool:
crane manifest docker.io/library/postgres:16     # exits non-zero if the manifest doesn't exist
crane digest   docker.io/library/postgres:16     # the immutable digest for that tag

# Without crane, plain docker is enough:
docker manifest inspect ghcr.io/mithr4ndir/discord-alert-proxy:latest

# Or curl the registry API directly (Docker Hub example):
curl -sIL https://registry.hub.docker.com/v2/library/postgres/manifests/16 \
  -H 'Accept: application/vnd.docker.distribution.manifest.v2+json'
```

If the manifest call returns 404, the image does not exist. No need to wait for a pod to `ImagePullBackOff` to find out.

## Things that look obvious but are not

- `ssh foo 'sudo bar'` will fail silently if sudo wants a password. Always use `BatchMode=yes` in scripts so it errors out.
- Heredocs over SSH are a footgun: `ssh foo <<EOF` runs locally if you do not quote `EOF`. Quote it: `<<'EOF'`.
- `cp -a` preserves mode/owner/timestamps. Use it for backups so the file does not get root-owned by accident.
- Always pipe through `--line-buffered` when greping a streaming log, otherwise blocks of output appear minutes late.
- `curl -f` treats `401` as failure. For a "service is up but rejecting unauth" probe, check `%{http_code}` directly: `curl -s -o /dev/null -w "%{http_code}" <url>`.
