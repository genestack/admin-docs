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
