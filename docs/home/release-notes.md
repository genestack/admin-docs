# Release notes

## Version 1.65

!!! tip ""
    Helm chart version 1.65.0

### Solr 10

ODM now runs on Solr 10. Solr 10 opens the search index that Solr 9 wrote, so no reindexing is needed.

!!! danger

    The upgrade to Solr 10 is one-way. Take a snapshot of the Solr volume before you upgrade. Once Solr 10 has
    written to the index, Solr 9 can no longer open it, so rolling back to an earlier ODM version means restoring
    that snapshot.

### Solr metrics without solr-exporter

Solr 10 no longer ships a Prometheus exporter, so the `solr-exporter` subchart is removed. Solr now serves its own
metrics in Prometheus format. To have the Prometheus operator scrape them, replace the `solr-exporter` section of
your values with:

```yaml
solr:
  serviceMonitor:
    enabled: true
```

The chart does not render values that still contain a `solr-exporter` section, and its error message points to
`solr.serviceMonitor`.

The metric names change as well: Solr 10 reports them in the OpenTelemetry style, for example
`solr_core_indexsearcher_index_num_docs` and `solr_core_index_size_megabytes`, with the core as a label. Update
dashboards and alerts that use the metric names of the old exporter.

### Import jobs run in applications, without func-job

`func-job` is no longer deployed on its own: the `applications` service now runs the import jobs. The `func-job`
pod, its Service and its ServiceMonitor are gone. The import jobs still keep their state in the `func_job` MySQL
database, and still read files through rclone.

The chart does not render values that still contain a `funcJob` section, and its error message says where each
value moves. Move what you set under `funcJob.files."/app/config/application.yaml"` into
`applications.files."/var/lib/genestack/properties/application.yaml"`, then delete the `funcJob` section:

- With an external MySQL, `spring.datasource` (`url`, `username`, `password`) becomes `frontend.mysql.funcJob`.
- `genestack.batch.*` becomes `func.job.batch.*`: `executorConcurrencyLimit`, `metadataItemChunkSize` and
  `cellsChunkSize`.
- `genestack.frontend.connection_timeout` becomes `func.job.callbackTimeout`.

For example, with an external MySQL:

```yaml
applications:
  files:
    "/var/lib/genestack/properties/application.yaml":
      frontend:
        mysql:
          funcJob:
            url: "jdbc:mysql://<EXTERNAL_MYSQL_HOST>/func_job?createDatabaseIfNotExist=true&permitMysqlScheme"
            username: "<EXTERNAL_MYSQL_USER>"
            password: "<EXTERNAL_MYSQL_PASSWORD>"
```

The chart's default `applications.resources` already include what `func-job` had by default. If you override
`applications.resources`, add to them what you gave `funcJob.resources`. Settings of the `funcJob` section such as
the affinity, node selector, probes or ServiceMonitor have no counterpart to move: those of `applications` now
apply to the import jobs.

Update dashboards and alerts that use the `func-job` pod, its metrics or its logs, so that they watch `applications`
instead: the import jobs run in its JVM, are scraped through its ServiceMonitor, and log to the `applications`
container. A log query on `container="func-job"`, in Loki for example, finds nothing new.

The number of import jobs is capped. `applications` runs at most `executorConcurrencyLimit` import jobs at once
(default 5), and queues up to `executorQueueCapacity` more (default 100). When the queue is full, a request that
starts or restarts an import gets HTTP 503 with the message "Too many imports are in progress: N are running and M
are waiting. Please try again later." The limit is what bounds the import jobs' memory in `applications`, so raise
it only together with `applications`' memory:

```yaml
applications:
  files:
    "/var/lib/genestack/properties/application.yaml":
      func:
        job:
          batch:
            executorConcurrencyLimit: 5
            executorQueueCapacity: 100
```

### MCP server

The Helm chart can now run the ODM MCP server, which lets AI assistants that support the Model Context Protocol
search and read ODM for the user signed in to them. It is off by default. To switch it on, see
[Enable the MCP server](mcp-server/enable.md).

## Version 1.64

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.64.0

### Genestack audit logs migration from MySQL to Clickhouse

This release migrates MySQL `genestack_audit` schema to ClickHouse `audit.Event` table and
removes `genestack_audit` schema from MySQL upon successful completion. Migration component responsible for it
is planned for a removal in the next release. As a result, this ODM version cannot be skipped during upgrades.

### Excluding audit events from tracking

It is now possible to exclude specific audit events from tracking if they are not needed, which saves disk space.
For example, you can exclude events related to data reads.

A configuration example is available in the `examples` directory of the Helm chart.

### Audit log retention period

!!! danger

    The retention period cannot be changed after installation. Set it before you install or upgrade to this version.

ODM keeps audit events in ClickHouse for a limited number of months and deletes older events automatically.
The default retention period is 12 months. To use a different one, set it in the Helm chart values:

```yaml
core:
  files:
    "/var/lib/genestack/properties/application.yaml":
      backend:
        clickhouse:
          audit:
            retentionMonths: 24
```
