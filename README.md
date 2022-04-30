Monitoring Stack
----------

## Prometheus

All metrics go into Prometheus, which has a 4 week retention period. Metrics in the `ha` namespace
are forwarded onto [Victoria Metrics](./victoria-metrics) for long term storage.

### Scrape Config

A prometheus `scrape_config` will look roughly like the following. In every case a new label
`job` is added to every metric coming in via a scrape target, which has the value of the `job_name`.
```
scrape_configs:
  - job_name: 'ha'
    metrics_path: /api/prometheus
    static_configs:
      - targets: ['home.mafro.net:2021']
```
