# Release notes

## Version 1.63

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.63.0

### GenestackRun-related tables migration from MySQL to Clickhouse

This release migrates the `GenestackRunRecord` and `GenestackSampleRunLink` tables from MySQL
to ClickHouse. As a result, this ODM version cannot be skipped when upgrading to a later version.
