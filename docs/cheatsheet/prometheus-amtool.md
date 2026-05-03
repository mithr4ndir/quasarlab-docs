# Prometheus and amtool

## promtool: validate before pushing

```bash
promtool check config /etc/prometheus/prometheus.yml
promtool check rules  /etc/prometheus/rules/*.yml
promtool query instant http://192.168.1.230:9090 'up{job="kubelet"} == 0'
promtool query range   http://192.168.1.230:9090 \
  --start=$(date -d '1 hour ago' -Iseconds) --end=$(date -Iseconds) --step=60 \
  'rate(node_cpu_seconds_total{mode="idle"}[1m])'

# amtool also has a check-config:
amtool check-config /etc/alertmanager/alertmanager.yml
```

The CI runs `promtool check rules` on every PR that touches `monitoring/`. Catches typos and bad PromQL before they reach the cluster.

## PromQL idioms I keep needing

```promql
# Targets that are down:
up == 0

# Pods that have restarted in the last 15 minutes:
increase(kube_pod_container_status_restarts_total[15m]) > 0

# Top 10 namespaces by CPU usage:
topk(10, sum by (namespace) (rate(container_cpu_usage_seconds_total{namespace!=""}[5m])))

# Alert on saturation: disk filling fast (will be full in <4h at current rate):
(predict_linear(node_filesystem_avail_bytes[1h], 4*3600) < 0)
  and node_filesystem_avail_bytes > 0

# Memory pressure approaching limit, per pod:
(container_memory_working_set_bytes{container!=""} /
 on(pod, container) kube_pod_container_resource_limits{resource="memory"})
> 0.9
```

## amtool: silence routine work

```bash
amtool --alertmanager.url=http://192.168.1.233:9093 alert query

# Silence a single alert for 2 hours:
amtool silence add alertname=KubeNodeNotReady instance=k8cluster1 \
  --duration=2h --comment="planned maintenance"

# List active silences:
amtool silence query

# Expire one early:
amtool silence expire <silence-id>
```

## Alertmanager routing recipe

The shape that has worked for me:

```yaml
route:
  group_by: [namespace, alertname]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: discord-default
  routes:
    - matchers: [severity = "critical"]
      receiver: discord-ops
      repeat_interval: 30m
    - matchers: [namespace = "media"]
      receiver: discord-media
    - matchers: [alertname =~ "Falco.*"]
      receiver: discord-security
```

The custom Discord proxy lives in `monitoring/discord-alert-proxy` and translates webhook payloads into channel-appropriate messages.

## Textfile collector (export from a script)

When the data you want to alert on is not exposed by an existing exporter, write it to a file that node_exporter's textfile collector picks up. Used in the lab for: ETCD DB size from the weekly maintenance run, 1Password rate-limit gauges, ansible-playbook-changed counts, `apt_upgrades_pending_total`.

```bash
# Standard layout: one file per metric source, .prom extension.
TEXTFILE_DIR=/var/lib/node_exporter/textfiles

# Atomic write (rename is atomic, write-and-rename avoids partial reads
# during scrapes):
TMPFILE=$(mktemp -t metric.XXXXXX)
cat > "$TMPFILE" <<EOF
# HELP my_metric A description scraped by node_exporter.
# TYPE my_metric gauge
my_metric{label="foo"} 42
my_metric_last_run_timestamp_seconds $(date +%s)
EOF
chmod 0644 "$TMPFILE"
mv -f "$TMPFILE" "$TEXTFILE_DIR/my_metric.prom"
```

Things to remember:

- One scrape interval (~30 sec) of latency between writing the file and seeing the value in Prometheus. Use `_last_run_timestamp_seconds` so you can alert on stale collectors.
- node_exporter must be running with `--collector.textfile.directory=$TEXTFILE_DIR`. Default on most distros.
- File ownership matters: writer must be able to overwrite, reader (node_exporter) must be able to read. The lab uses `0644` files in a `0755` directory.

## Things that are easy to get wrong

- **`for: 5m` matters.** A bare alert expression fires on the first scrape. Always set `for:` so transient blips do not page.
- **Use `absent()` for "this should be reporting":**

```promql
absent(up{job="postgres-exporter"})
```

  This catches "the exporter itself is gone," which a `up == 0` alert cannot, because if the target does not exist there is nothing to be `0`.
- **Cardinality is real.** Labels with unbounded values (request URLs, user IDs, container UIDs) blow up Prometheus memory. Drop them at the scrape config or the relabel level.
