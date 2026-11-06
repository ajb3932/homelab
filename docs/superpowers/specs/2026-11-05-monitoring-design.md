# Monitoring stack (Loki, Alloy, Prometheus, Grafana)

## Why

On 20 Oct 2026 homelab ran out of memory, hung, and had to be hard rebooted. Nothing recorded what was eating RAM beforehand. After the next crash it should be possible to open Grafana and see what happened in the minutes before the hang: RAM and swap, which processes and containers were growing, kernel messages (OOM killer, hung tasks), and container logs.

## Goals

- Record host RAM/swap/cache, per-container memory and CPU, per-process memory and CPU (grouped by process name).
- Record all container logs (labelled by container and compose stack) and the host systemd journal (including kernel messages, labelled so OOM/kernel lines are easy to filter).
- Data survives a hard reboot: all storage on disk under `/lab/appdata/monitoring/`; Alloy resumes collecting from where it stopped.
- One provisioned Grafana dashboard, "Homelab health": memory and swap over time, top processes by memory, top containers by memory, kernel/OOM log panel.

## Non-goals

- No SWAG or public access. Grafana is local only at `homelab:3000`.
- No syslog server or rsyslog. The journal is already persistent (5 boots on record).
- No alerting for now.

## Anti-goals

- The monitoring stack must not make memory pressure worse. Every container has a memory limit.

## Decisions

- Stack folder `monitoring/compose.yaml`, deployed as a Portainer git stack from `ajb3932/homelab` main (public repo), like every other stack, endpoint `homelab` (id 4), 5 minute auto-update.
- Config files (Loki, Alloy, Prometheus, Grafana provisioning, dashboard) live in `monitoring/` in the repo and are embedded in the compose file using compose `configs:` with inline `content:` (Portainer's compose is v2.40.3, which supports it). This keeps configs in git without relying on relative bind mounts, which are unreliable for Portainer git stacks.
- Images pinned to current stable versions: `grafana/loki`, `grafana/alloy`, `prom/prometheus`, `grafana/grafana`.
- Data under `/lab/appdata/monitoring/{loki,prometheus,grafana,alloy}` (gets picked up by the existing borg backup of `/lab/appdata`).
- Retention 30 days for logs (Loki) and metrics (Prometheus).
- Memory limits: Loki 1 GB, Prometheus 1 GB, Grafana 512 MB, Alloy 512 MB.
- Alloy collects everything: `discovery.docker` + `loki.source.docker` for container logs; `loki.source.journal` for the journal; `prometheus.exporter.unix` (host), `prometheus.exporter.cadvisor` (containers) and `prometheus.exporter.process` (processes), pushed to Prometheus via remote write. Alloy runs with host PID namespace and read-only mounts of `/proc`, `/sys`, `/`, `/var/log/journal`, `/run/log/journal`, `/etc/machine-id`, and the Docker socket.
- Prometheus has the remote-write receiver enabled and does no scraping of its own.
- Grafana admin password comes from a `GF_ADMIN_PASSWORD` Portainer stack variable, never committed. Data sources (Loki, Prometheus) and the dashboard are provisioned.
- The stack uses its own default network. Nothing joins `proxy-net`.

## Changes made during deployment

- cadvisor uses `docker_only = false` (with `store_container_labels = false`). With `true` it silently reported no containers on this host. Alloy also mounts the Docker root at its real path (`/lab/docker/docker`) and `/run/containerd/containerd.sock`.
- On first start Alloy backfills the last 30 days of the journal (about 20 minutes). Container log lines older than 30 days are rejected by Loki, which is expected.
- Alloy at 512 MB was OOM-killed 17 times in its first 24 hours and swapped heavily inside its cgroup. Its limit is now 1 GB with `memswap_limit: 1g` (no swap) and `GOMEMLIMIT=800MiB`, and cadvisor's `container_label_*` labels are dropped before remote write.
- Alerting: Grafana file-provisioned contact point (webhook to ntfy `ajb-monitor?template=grafana`, Bearer token from the `NTFY_TOKEN` stack variable), with the rule "RAM above 95%" for 2 minutes. Grafana expands env vars in provisioning files, so templates use `.Values`, never `$values`.

## Left to the builder

- Exact dashboard panel queries and layout.
- Process grouping rules for the process exporter (default: group by process name).
