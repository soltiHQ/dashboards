# Solti - Agent Overview

Single-pane Grafana dashboard for [Solti](https://github.com/soltiHQ) task-orchestration agents. 
Shows fleet health, workload phase distribution, task lifecycle, admission control, supervisor event bus, API surface, control-plane discovery, and host process metrics: all from one Prometheus scrape.

## What you'll see

- **Health (always open).** Live agent count, task throughput, p95 task duration, error rate.
- **Fleet (always open).** Per-agent table: Ready/NotReady state, in-flight tasks, throughput, 5-min success rate, time since last successful control-plane sync. Per-agent up/down state-timeline next to it.
- **Workload (always open).** Current task distribution across phases (`pending`, `running`, `succeeded`, `failed`, `timeout`, `canceled`, `exhausted`) as a donut + stacked timeline of how that distribution evolves.

**Drill-down rows (collapsed by default).**
- **Activity**: task duration heatmap + p50/p95/p99 percentiles.
- **Event bus**: supervisor event throughput by kind + lost-events counter.
- **API**: request rate by status (HTTP / gRPC), latency heatmap.
- **Lifecycle**: backoff cycles by source (success / failure), terminal states & timeouts. Restart is **not** an error — it's a normal supervisor cycle for periodic tasks.
- **Discovery**: heartbeat rate by outcome, failure reasons.
- **Admission**: submitted vs rejected (diverging bars), rejection reasons (`slot_full`, `slot_busy`, `add_failed`, `bus_lagged`, …).
- **Host**: process CPU, RSS, file descriptors, uptime. (linux only)

## Where the metrics come from

The dashboard consumes Prometheus metrics published by the [`solti-prometheus`](https://github.com/soltiHQ/sdk/tree/main/crates/solti-prometheus) crate of the Solti SDK. Wire it into your agent:

```rust
use std::sync::Arc;
use solti_prometheus::{
    PrometheusMetrics, PrometheusSubscriber,
    PrometheusApiMetrics, PrometheusDiscoverMetrics, PrometheusStateCollector,
    Registry, register_build_info, register_process_collector,
};

let registry = Arc::new(Registry::new());

// Core: always wire these.
let metrics    = PrometheusMetrics::new(registry.clone())?;
let subscriber = PrometheusSubscriber::new(registry.clone())?;
register_build_info(&registry, &[
    ("agent",   "agentd-http"),
    ("version", env!("CARGO_PKG_VERSION")),
])?;

// Optional but recommended (each enables specific dashboard rows):
let state_collector = PrometheusStateCollector::new(supervisor_api.state())?;
registry.register(Box::new(state_collector))?;
register_process_collector(&registry)?;
let api_metrics      = Arc::new(PrometheusApiMetrics::new(registry.clone())?);
let discover_metrics = Arc::new(PrometheusDiscoverMetrics::new(registry.clone())?);

// Then expose /metrics and let Prometheus scrape it.
```

Full reference agent: [`examples/agentd-http`](https://github.com/soltiHQ/sdk/tree/main/examples/agentd-http) and [`examples/agentd-grpc`](https://github.com/soltiHQ/sdk/tree/main/examples/agentd-grpc).

## Required Prometheus metrics

| Metric prefix                | Source                                                       | Powers                                            |
|------------------------------|--------------------------------------------------------------|---------------------------------------------------|
| `solti_runner_*`             | `PrometheusMetrics`                                          | Health stats, Activity, Event bus                 |
| `solti_sv_*`, `solti_ctrl_*` | `PrometheusSubscriber`                                       | Workload, Lifecycle, Admission, Event bus         |
| `solti_sv_tasks_by_phase`    | `PrometheusStateCollector` (feature `state`)                 | Workload phase donut & timeline                   |
| `solti_api_*`                | `PrometheusApiMetrics` (feature `api`)                       | API row                                           |
| `solti_discover_*`           | `PrometheusDiscoverMetrics` (feature `discover`)             | Discovery row, Fleet "Last sync" column           |
| `process_*`                  | `register_process_collector` (feature `process`, Linux only) | Host row                                          |
| `solti_build_info`           | `register_build_info`                                        | `$agent` variable, Fleet table identity columns   |
| `up{job="solti-agent"}`      | Prometheus built-in                                          | Fleet State column, Availability timeline         |

Missing one of these makes the corresponding row empty — the dashboard does not error out.

## Variable

- `$agent` *(multi-select, default `All`)*: filters every panel by agent type (`agentd-http`, `agentd-grpc`, your custom build, …). 
Populated from `label_values(solti_build_info, agent)`.

## Setup notes

- Use scrape `job_name: 'solti-agent'` in your Prometheus config — the **State** column and **Availability** timeline rely on the `up{job="solti-agent"}` label.
- Optional `host` label on each scrape target (set in `static_configs.labels.host`) populates the Hostname column. Useful when you run multiple agents.

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'solti-agent'
    static_configs:
      - targets: ['agent-1.lan:9090']
        labels: { host: 'agent-1' }
      - targets: ['agent-2.lan:9090']
        labels: { host: 'agent-2' }
```

## Source & changelog

Dashboard JSON, change history, and additional Solti dashboards live at <https://github.com/soltiHQ/dashboards>.

Solti SDK source: <https://github.com/soltiHQ/sdk>.

## License

Apache-2.0
