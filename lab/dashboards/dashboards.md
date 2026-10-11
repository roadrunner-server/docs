# RoadRunner Dashboards

Grafana dashboards for RoadRunner installations, including Temporal. The available metrics are described in [Metrics](../metrics.md).

Each dashboard is a Grafana export with a `DS_PROMETHEUS` datasource input. Import the JSON in Grafana and select the Prometheus datasource that scrapes the `metrics.address` endpoint of the [metrics plugin](../metrics.md). The uptime, heap, and goroutine panels use the `process_*` and `go_*` [runtime metrics](../metrics.md#runtime-metrics).

Each dashboard has a "Workers over time" panel with the total, ready, and working worker counts. With [automatic scaling](../../php/auto-scaling.md), the total count rises above `num_workers` during scale-up and returns to it after the idle timeout.

## gRPC dashboard

This dashboard contains gRPC plugin metrics: [gRPC dashboard JSON](./grpc.md)

## HTTP dashboard

This dashboard contains http plugin metrics: [HTTP dashboard JSON](./http.md)

The request, latency, queue, and "No free workers" panels need the `http_metrics` middleware in `http.middleware`. See [HTTP Metrics](../metrics.md#http-metrics).

## JOBS dashboard

This dashboard contains jobs plugin metrics: [JOBS dashboard JSON](./jobs.md)

The rate panels use the v6 counters `rr_jobs_jobs_ok`, `rr_jobs_jobs_err`, `rr_jobs_jobs_requeue`, and `rr_jobs_push_err`. See [JOBS Metrics](../metrics.md#jobs-metrics).

## Temporal dashboard

This dashboard contains the RoadRunner part of the Temporal plugin metrics: [Temporal dashboard JSON](./temporal.md). It covers the activity worker pool and the `rr_activities_pool_queue_size` and `rr_workflows_pool_queue_size` gauges. The queue size gauges are served on `temporal.metrics.prometheus.address`, not on `metrics.address`. Add that endpoint as a scrape target of the same Prometheus datasource. The panel queries use the gauge names without a `prefix`.

Temporal publishes SDK dashboards in the [temporalio/dashboards](https://github.com/temporalio/dashboards) repository. RoadRunner emits SDK metrics through Tally, so the matching file is [sdk/temporal-go-java-sdks-tally.json](https://github.com/temporalio/dashboards/blob/master/sdk/temporal-go-java-sdks-tally.json). The file does not match the RoadRunner v6 metrics without changes:

- Its queries use the Prometheus naming convention with `_total` and `_seconds` suffixes, for example `temporal_request_total` and `temporal_request_latency_seconds_bucket`. RoadRunner emits `temporal_request` and `temporal_request_latency`. Remove the suffixes from the queries.
- Its latency panels use `histogram_quantile()` on `_bucket` series. Set `temporal.metrics.prometheus.type: histogram` to export them. The default `summary` type exports `quantile` series instead. See [Temporal Metrics](../metrics.md#temporal-metrics).
