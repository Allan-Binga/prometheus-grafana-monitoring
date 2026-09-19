# Monitoring

Shared Prometheus and Grafana stack for application and Linux host metrics. One Prometheus instance scrapes multiple targets; Grafana queries that instance to display dashboards.

```text
Node.js application :5900 ──┐
                           ├── Prometheus :9090 ── Grafana :3000
Linux / Node Exporter :9100 ┘
```

## Repository layout

```text
monitoring/
├── docker-compose.yml          # Prometheus, Grafana, and Node Exporter
├── prometheus/
│   └── prometheus.yml          # Scrape targets and collection intervals
└── grafana/
    └── dashboards/
        └── node-js.json        # Importable Node.js dashboard
```

`production.txt` contains local deployment notes and is ignored by Git. It is not loaded by Docker Compose or Prometheus. Production changes must be applied to the actual configuration files.

## Current configuration

| Service | Pinned image | Purpose |
| --- | --- | --- |
| Prometheus | `prom/prometheus:v3.14.0` | Collects and stores metrics |
| Grafana | `grafana/grafana:13.2.2` | Displays metrics and dashboards |
| Node Exporter | `quay.io/prometheus/node-exporter:v1.12.1` | Exposes Linux host metrics |

Prometheus scrapes every 15 seconds. The active jobs are:

| Job | Scrape address | Metrics path | Instance label |
| --- | --- | --- | --- |
| `prestige-hostel-local` | `host-gateway:5900` | `/prestige-hostel/v1/metrics` | `localhost:5900` |
| `node-exporter` | `host-gateway:9100` | `/metrics` | `host-gateway:9100` |

`host-gateway` is configured in Compose to resolve to the Docker host. Relabeling the app instance to `localhost:5900` only changes its label; it does not change the connection address.

## Start locally

Requirements: a Linux host, Docker Engine, and the Docker Compose plugin. The host application must already expose its metrics endpoint and be reachable from the Docker network. A service bound only to host loopback (`127.0.0.1`) will not be reachable through the bridge gateway.

From this repository:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

The stack does not start the Node.js application. Stop any older monitoring stack occupying ports 3000 or 9090 first. The explicit container name `prometheus` must also be available; a stopped container with that name still conflicts. Node Exporter uses host port 9100 directly.

Open:

- Grafana: <http://localhost:3000>
- Prometheus: <http://localhost:9090>
- Prometheus targets: <http://localhost:9090/targets>

With a fresh Grafana database, the current configuration initializes the login as `admin` / `admin`. Change this before production. Changing the initialization environment variable does not reset a password in an existing database.

## Connect Grafana and import dashboards

1. In Grafana, add a **Prometheus** data source.
2. Set the URL to `http://prometheus:9090` and select **Save & test**.
3. In Explore, select that data source and query `nodejs_version_info`.
4. Import `grafana/dashboards/node-js.json` for the application dashboard.
5. Import dashboard **1860 — Node Exporter Full** for the Linux host. Select your Prometheus data source, the `node-exporter` job, and `host-gateway:9100` instance.

Inside the Grafana container, `localhost` means Grafana itself. Use the Compose service name `prometheus` to reach Prometheus.

The saved Node.js dashboard references data-source UID `dfymrlzr4x3i8b`. A fresh Grafana installation may generate a different UID. Replace that UID throughout the JSON with the new one before importing, including references in query targets and the instance variable. The UID is visible in the data-source settings URL after `/edit/`. Selecting a default data source does not override explicit dashboard references.

Dashboard files are imported manually: this Compose configuration does not mount them or configure Grafana provisioning. Editing JSON on disk requires reimporting it; overwrite the existing dashboard to apply changes. Some Node Exporter Full panels require optional collectors; this stack enables the default collectors only.

## Verify collection

Run these queries in Prometheus or Grafana Explore:

```promql
up
nodejs_version_info{job="prestige-hostel-local"}
node_uname_info{job="node-exporter"}
```

`up = 1` means the scrape succeeded; `up = 0` means it failed. An empty result can mean the job has not been loaded or scraped yet. Allow several scrapes for rate charts to populate.

The Node.js version is a label on `nodejs_version_info`; the metric value itself is `1`. The version Stat panel should use an instant query, legend `{{version}}`, text mode **Name**, and no restrictive field filter.

## Add another application or server

Each monitored application needs instrumentation and a metrics endpoint. Each additional Linux server needs its own Node Exporter. Adding a scrape job does not install these components.

A second host application could be added under `scrape_configs`:

```yaml
  - job_name: 'second-app'
    metrics_path: /metrics
    static_configs:
      - targets: ['host-gateway:6000']
```

A separate Linux server reachable over a private network or VPN could be added as:

```yaml
  - job_name: 'node-exporter-remote'
    static_configs:
      - targets: ['10.20.0.10:9100'] # Replace with the actual private address.
```

For a container on the same Docker network, use its service name and internal port. Containers in separate Compose projects need a shared network or another reachable address. For PostgreSQL and other databases, run the corresponding exporter and scrape its HTTP metrics endpoint, not the database protocol port.

After editing `prometheus/prometheus.yml`, validate and reload:

```bash
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
curl --fail --request POST http://localhost:9090/-/reload
```

Reload only if validation succeeds. The lifecycle endpoint is enabled in Compose. Check Targets afterward. If a file replacement leaves the container seeing an old bind-mounted configuration, recreate Prometheus with `docker compose up -d --force-recreate prometheus`.

## How Linux monitoring works

Node Exporter uses host networking and the host PID namespace. The read-only mount `/:/host:ro,rslave` and `--path.rootfs=/host` let it inspect host filesystems. This configuration monitors the Linux host running Docker and exposes its CPU, memory, disk, and network metrics on port 9100.

Because it uses host networking, there is no Compose `ports` mapping for Node Exporter. Its listener must be protected at the host/network level in production.

## Prepare for production

The checked-in Compose file is a local starting point: Grafana uses the initial password `admin`, ports 3000 and 9090 bind publicly on the host, and Node Exporter listens on host port 9100. Apply the following to your deployment before exposing the VPS.

### Network and access

For a stack and application on the same VPS, the existing `host-gateway` targets can stay unchanged. They resolve to that VPS. Rename the application job and instance labels if desired, and update any dashboard filters selecting their old values.

Keep Prometheus and metrics endpoints private. For a host-based HTTPS reverse proxy or SSH tunnel, replace the published port mappings with:

```yaml
# Under prometheus:
ports:
  - "127.0.0.1:9090:9090"

# Under grafana:
ports:
  - "127.0.0.1:3000:3000"
```

A containerized reverse proxy should reach Grafana at `grafana:3000` over a shared Docker network instead. Configure HTTPS for the real Grafana domain. Restrict port 9100 and the app's direct metrics access to trusted monitoring traffic. Block the application's metrics path in its public reverse proxy. Verify restrictions externally; Docker-published ports can interact differently with firewall rules than ordinary host services.

For remote targets, use a private network/VPN, or explicitly configure authenticated TLS. Do not simply expose exporter ports on the public internet.

### Grafana settings

Replace the existing Grafana environment block and add its restart policy:

```yaml
restart: unless-stopped
environment:
  GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:?Set GRAFANA_ADMIN_PASSWORD}
  GF_USERS_ALLOW_SIGN_UP: "false"
  # Enable after configuring the actual HTTPS reverse proxy:
  # GF_SERVER_ROOT_URL: https://grafana.example.com
  # GF_SECURITY_COOKIE_SECURE: "true"
```

Provide the password through deployment configuration or an untracked `.env` file. Add `.env` to `.gitignore` before creating it; the current ignore file only lists `production.txt`. The password variable initializes a fresh database only.

### Storage and operations

Keep the named volumes. Optionally append these flags to Prometheus's existing `command` list, sizing them for the VPS:

```yaml
- '--storage.tsdb.retention.time=30d'
- '--storage.tsdb.retention.size=5GB'
```

These are example limits, not a strict cap on all disk usage. Leave headroom for the write-ahead log, current data, and other services. Whichever retention limit is reached first triggers retention cleanup.

Keep image versions pinned and upgrade deliberately. Back up Grafana's database and retain dashboard exports; use a supported snapshot or stopped-service backup process for Prometheus data. Configure alerts separately for outages and resource exhaustion: this repository currently has no alert rules or notification routing. Monitoring on the same VPS cannot report its own complete outage without an external check.

## Volumes and routine commands

- `prometheus_data` stores metrics history.
- `grafana_data` stores Grafana's database, dashboards, users, and data-source configuration.

Compose normally prefixes these volume names with its project name. Moving or renaming the project can select new volumes. Starting fresh is fine, but existing data will not migrate automatically.

```bash
# Service logs
docker compose logs --tail=100 prometheus grafana node-exporter

# Stop the stack while retaining its named volumes
docker compose down

# Apply Compose changes
docker compose up -d
```

Do not use `docker compose down -v` unless you intend to delete the stack's stored metrics and Grafana data.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Prometheus is green but Grafana has no data | Query in Explore; verify panel and variable data-source UIDs, instance filters, and time range. |
| Grafana cannot connect to Prometheus | Use `http://prometheus:9090` and confirm both services share the Compose network. |
| Application target is DOWN | Check app process, port, metrics path, bind address, and firewall. Read the target's scrape error. |
| Mount error says file versus directory | Confirm `./prometheus/prometheus.yml` exists as a file before starting Compose. |
| New target is absent | Validate and reload Prometheus, then wait for a scrape. |
| Dashboards disappeared after moving repositories | Check which Compose project and Grafana volume are in use. |
| Container name or port already in use | Check older monitoring containers; stop the conflicting stack before starting this one. |

## References

- [Node Exporter host monitoring](https://github.com/prometheus/node_exporter#docker)
- [Prometheus security model](https://prometheus.io/docs/operating/security/)
- [Grafana Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/)
- [Node Exporter Full dashboard](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
