# Solti Dashboards

Grafana dashboards for the [Solti](https://github.com/soltiHQ) project ecosystem.

## Catalog

[soltiHQ/sdk](https://github.com/soltiHQ/sdk): task-orchestration agents. Metrics produced by `solti-prometheus`.

| Dashboard            | Source file                  | Metrics from                      | UID                    | grafana.com ID |
|----------------------|------------------------------|-----------------------------------|------------------------|----------------|
| Solti Agent Overview | `solti/agent-overview.json`  | `solti-prometheus` (SDK >= 0.0.2) | `solti-agent-overview` | _TBD_          |

## Usage

### Import via Grafana UI

1. Open your Grafana → **Dashboards → New → Import**
2. Either paste the dashboard ID from the catalog (once published to `grafana.com`), or upload the JSON file from `solti/`
3. Select your Prometheus data source when prompted
4. Done

### Local provisioning (Docker / Kubernetes)

Mount the JSON files into Grafana's provisioning directory:

```yaml
# docker-compose.yml
services:
  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./dashboards/solti:/var/lib/grafana/dashboards/solti:ro
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
```

```yaml
# grafana/provisioning/dashboards/solti.yaml
apiVersion: 1
providers:
  - name: 'Solti'
    folder: 'Solti'
    type: file
    options:
      path: /var/lib/grafana/dashboards/solti
```

When provisioning, replace `${DS_PROMETHEUS}` with your actual data-source UID (e.g. `prometheus`). 

## License

[Apache License, Version 2.0](LICENSE)

## Contributing

Found a bug? Have an idea? [Open an issue](https://github.com/soltiHQ/taskvisor/issues) or send a pull request.
