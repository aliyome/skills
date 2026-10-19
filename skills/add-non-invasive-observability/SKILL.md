---
name: add-non-invasive-observability
description: >-
  Non-invasive observability stack addition for projects using mise and docker-compose.
  Use when adding Grafana LGTM, Alloy, or eBPF auto-instrumentation without touching application code,
  generating mise observability tasks, or diagnosing container telemetry pipelines.
---

# Non-invasive observability with mise and docker-compose

Add full-stack observability (Metrics, Logs, Traces, Profiles) to an existing docker-compose application managed by mise, with **zero application code changes**.

## Principles

- **Zero code changes**: Application source code, Dockerfiles, and runtime dependencies remain untouched. All instrumentation attaches via eBPF, network proxies, or container runtime sockets.
- **Overlay isolation**: Keep base compose files clean. Place monitoring services in an overlay file (`compose.observability.yml`).
- **Mise as single entrypoint**: Expose operational workflows (`obs:up`, `obs:down`, `obs:status`, `obs:verify`) through `mise.toml` tasks.

## Workflow

### 1. Inspect existing environment
Inspect services, ports, networks, and task management.
- Run `docker compose config --services` to discover active service names.
- Run `docker compose config` to inspect the target application container name, listening internal ports, and docker network names.
- Check port availability on host: Grafana (3000), Alloy (12345), OTLP (4317/4318), DB exporter (9104).
- Inspect database and cache services (e.g. MySQL, PostgreSQL, Redis) to select corresponding exporters.
- Inspect `mise.toml` (or create if missing) to check existing task conventions.

**Completion criterion**: A recorded target profile documenting:
1. Target application service name & internal listening port (e.g., `app`, port `8080`)
2. Shared docker network name (e.g., `default` or `my_network`)
3. Database type and credentials for exporter sidecar
4. Absence of host port conflicts for `3000` and `12345`

### 2. Scaffold observability directory structure
Create isolated observability configuration directories alongside compose files:
- `observability/alloy/config.alloy`
- `observability/obi/obi-config.yaml`
- `observability/grafana/datasources.yaml`
- `compose.observability.yml`

Populate configs using the templates in [references/](references/):
- Adapt [compose.observability.yml](references/compose.observability.yml) with your target network and app service name.
- Adapt [config.alloy](references/config.alloy) with appropriate exporter targets.
- Adapt [obi-config.yaml](references/obi-config.yaml) targeting the application listening port.
- Adapt [datasources.yaml](references/datasources.yaml) with datasource correlation rules.

**Completion criterion**: `docker compose -f <base-compose> -f compose.observability.yml config` exits 0.

### 3. Wire mise tasks
Add observability management tasks to `mise.toml` using [references/mise.tasks.toml](references/mise.tasks.toml) as a reference:
- `obs:up`: Start base services + observability overlay in detached mode.
- `obs:down`: Stop observability overlay and base services.
- `obs:logs`: Follow logs for alloy, otel-lgtm, and obi.
- `obs:verify`: Run automated health checks against endpoints.

**Completion criterion**: `mise tasks ls` lists all `obs:*` tasks without syntax errors.

### 4. Verify & smoke test
Execute the pipeline verification:
1. Launch stack: `mise run obs:up`
2. Check container status: all services reporting `running` (Up).
3. Smoke test endpoints:
   - Grafana UI: `curl -sS -o /dev/null -w "%{http_code}" http://localhost:3000` returns `200`.
   - Alloy status: `curl -sS -o /dev/null -w "%{http_code}" http://localhost:12345/-/ready` returns `200`.
   - Prometheus metrics: verify target scrape status via `curl -sS "http://localhost:3000/api/datasources/proxy/uid/prometheus/api/v1/query?query=up"`.
4. Send tracer traffic to the application and confirm:
   - Tempo receives traces from OBI (`service.name` matches target app).
   - Loki collects container logs with parsed `trace_id` and `span_id`.
   - Tempo span detail links directly to Loki logs.

**Completion criterion**: Grafana responds with HTTP 200, Prometheus reports healthy targets, and Tempo indexes traces from the target service.

## Reference: eBPF Requirements & Constraints

eBPF instrumentation (OBI & Pyroscope eBPF) operates at the kernel level:
- Container capabilities: `privileged: true`, `pid: host`, and mount `/sys/fs/bpf:/sys/fs/bpf`.
- macOS / Windows Docker Desktop: Runs inside Docker Desktop's Linux VM. Ensure Docker Desktop has sufficient allocated memory (minimum 4GB recommended).
- Linux hosts: Requires Linux kernel version >= 5.8 with BTF enabled (`CONFIG_DEBUG_INFO_BTF=y`).
