Monitoring Stack
==========

A prometheus stack for a simple monitoring set up.


Updating
----------

Update the docker tag versions in [`.env`](./.env).

 * https://hub.docker.com/r/prom/prometheus/tags 
 * https://hub.docker.com/r/grafana/grafana/tags
 * https://hub.docker.com/r/prom/node-exporter

```
docker compose pull
docker compose up -d
```

Prometheus
----------

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

### Remote Write

It's easy to forward metrics onto another agent who supports the prometheus `remote_read` protocol.
The following relabel config keeps only metrics with a label of `job=ha`.

```
remote_write:
  - url: http://vmetrics:8428/api/v1/write
    write_relabel_configs:
      - source_labels: [job]
        regex: 'ha'
        action: keep
```

### Snapshot Prometheus data

As a prerequisite, Prometheus must be started with the Admin API enabled via `docker-compose.yml`:
```
command:
  - '--web.enable-admin-api'
```

Tell Prometheus to take a snapshot via its API:
```
> curl -XPOST http://ringil:9090/api/v1/admin/tsdb/snapshot
{"status":"success","data":{"name":"20220426T130745Z-09bdfe7dbe3e0c45"}}
```

See the snapshot on disk:
```
> docker compose exec prometheus ls snapshots/
20220426T130745Z-09bdfe7dbe3e0c45
```

Copy the snapshot from the docker volume to the host filesystem:
```
> docker compose cp prometheus:/prometheus/snapshots/20220426T130745Z-09bdfe7dbe3e0c45 .
```

### Import a snapshot into Prometheus

Import the snapshot into Victoria Metrics:
```
> docker run --rm -v $(pwd)/20220426T130745Z-09bdfe7dbe3e0c45:/import victoriametrics/vmctl prometheus --prom-snapshot /import --vm-addr http://192.168.1.198:8428 -s
Prometheus import mode
Prometheus snapshot stats:
  blocks found: 17;
  blocks skipped by time filter: 0;
  min time: 1650196803592 (2022-04-17T12:00:03Z);
  max time: 1650978445115 (2022-04-26T13:07:25Z);
  samples: 46209935;
  series: 13335.
.. snip ..
0 p/s2022/04/26 13:24:42 Import finished!
2022/04/26 13:24:42 VictoriaMetrics importer stats:
  idle duration: 18.779193147s;
  time spent while importing: 3m1.762186564s;
  total samples: 46209935;
  samples/s: 254232.94;
  total bytes: 930.8 MB;
  bytes/s: 5.1 MB;
  import requests: 230;
  import requests retries: 0;
2022/04/26 13:24:42 Total time: 3m1.862603292s
```
