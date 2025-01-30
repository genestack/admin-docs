# ClickHouse Rebalancing

Rebalancing shards in ClickHouse is primarily a manual process due to inherent [limitations](https://clickhouse.com/docs/en/guides/sre/scaling-clusters) in ClickHouse. To simplify this process, we have developed a tool to assist with shard rebalancing.

## Prerequisites

- Ensure there are no running ODM tasks. Wait for all tasks to complete before proceeding. This step is crucial to maintain data consistency in ClickHouse.
- Ensure that there is enough free space in the ClickHouse cluster. All rebalanced data should be distributed equally across the nodes.
- Make sure ODM version is 1.60 or higher.
- Make sure `clickhouse-helper` version is higher than 0.30.0.

## Just to be sure

You can use [sanity check](../troubleshooting/sanity-check.md) just to doublecheck that data is consistent in ODM.

## Steps for Rebalancing

### 1. Enable ClickHouse Read-Only Mode in ODM

Set ODM to read-only mode to prevent any write operations during the rebalancing process. This does not affect schema migrations.

```shell
export ODM_CORE_URL=http://<ODM_CORE_HOST>:<ODM_CORE_PORT>
docker run \
  --env ODM_CORE_URL=${ODM_CORE_URL} \
  091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/clickhouse-helper \
  odm readonly --set-value=true
```

### 2. Redeploy Services with the New ClickHouse Database

Update your Helm values to point to the new ClickHouse database and redeploy the `core` and `applications` services.

#### a) Update Helm Values

Refer to the example values file patch for guidance: [clickhouse-new-database.yaml](files/clickhouse-new-database.yaml). Use the `genestack_new` database name.

#### b) Perform Helm Upgrade

Run the following command to apply the changes:

```shell
helm upgrade <release-name> <chart-name> -f values.yaml
```

### 3. Clone Data to the New Database

Use the `clickhouse-helper` tool to copy data from the old database to the new one. Both `CH_SOURCE_URL` and `CH_DESTINATION_URL` can accept multiple nodes separated by a comma (`,`), for example, `localhost:9000,localhost:19000`. **It is recommended to include all nodes in the cluster**.

Follow these steps:

1. Set the source and destination ClickHouse server URLs:

    ```shell
    export CH_SOURCE_URL=<SOURCE_CLICKHOUSE_HOST>:<SOURCE_CLICKHOUSE_PORT>
    export CH_DESTINATION_URL=<DESTINATION_CLICKHOUSE_HOST>:<DESTINATION_CLICKHOUSE_PORT>
    ```

2. Set the source and destination database names:

    ```shell
    export CH_SOURCE_DATABASE=genestack
    export CH_DESTINATION_DATABASE=genestack_new
    ```

3. Run the `clickhouse-helper` to clone the data:

    ```shell
    docker run \
    --env CH_SOURCE_URL=${CH_SOURCE_URL} \
    --env CH_DESTINATION_URL=${CH_DESTINATION_URL} \
    --env CH_SOURCE_DATABASE=${CH_SOURCE_DATABASE} \
    --env CH_DESTINATION_DATABASE=${CH_DESTINATION_DATABASE} \
    091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/clickhouse-helper \
    ch clone
    ```

### 4. Disable ClickHouse Read-Only Mode in ODM

Once the data cloning is complete, re-enable write operations in ODM.

```shell
export ODM_CORE_URL=http://<ODM_CORE_HOST>:<ODM_CORE_PORT>
docker run \
  --env ODM_CORE_URL=${ODM_CORE_URL} \
  091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/clickhouse-helper \
  odm readonly --set-value=false
```

## Notes

- Ensure all steps are followed in sequence to avoid data inconsistencies.
- The `clickhouse-helper` tool is essential for simplifying the rebalancing process.
- Remember to delete the old database from ClickHouse after the rebalancing process is complete.

    ```shell
    clickhouse-client --host <SOURCE_CLICKHOUSE_HOST> --port <SOURCE_CLICKHOUSE_PORT> -q "DROP DATABASE genestack"
    ```
