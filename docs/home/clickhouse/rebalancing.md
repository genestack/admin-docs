# Clickhouse rebalancing

Clickhouse rebalancing is mostly manual process, due to clickhouse limitations.
Becasuse of this we developed a tool that will help you to make shard rebalancing easier.

## Sequence of actions

1) Make sure there is no ODM tasks in running state, wait until all of them finished. It's an important step to keep data consistent in Clickhouse.

2) Configure ODM to be in clickhouse read-only mode:

    ```shell
    export ODM_CORE_URL=http://<ODM_CORE_HOST>:<ODM_CORE_PORT>
    clickhouse-helper odm readonly --set-value=true
    ```

   Note: Read-only mode doesn't affect schema migration.

3) Redeploy `core` and `applications` services with new database in clickhouse.
   1) Update required values in helm values. View the values file patch [example](files/clickhouse-new-database.yaml) using `genestack_new` database name.
   2) Run helm upgrade.

4) Clone data from the previous database to the new one.

    ```shell
    export CH_SOURCE_URL=<SOURCE_CLICKHOUSE_HOST>:<SOURCE_CLICKHOUSE_PORT>
    export CH_DESTINATION_URL=<DESTINATION_CLICKHOUSE_HOST>:<DESTINATION_CLICKHOUSE_PORT>
    export CH_SOURCE_DATABASE=genestack
    export CH_DESTINATION_DATABASE=genestack_new
    clickhouse-helper ch clone
   ```

5) Configure ODM to be in clickhouse read-write mode:

    ```shell
    export ODM_CORE_URL=<ODM_CORE_HOST>:<ODM_CORE_PORT>
    clickhouse-helper odm readonly --set-value=false
    ```
