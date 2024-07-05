# Sanity check using description

1. Gather required variables.

    ```shell
    export ODM_URL=https://odm.localhost
    export ODM_TOKEN=yourTokenFromRootUser
    ```

2. Run check over ODM instance before the upgrade.

    ```shell
    docker run \
      --env ODM_URL=${ODM_URL} \
      --env ODM_TOKEN=${ODM_TOKEN} \
      091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/clickhouse-helper \
      odm sanity > sanity-result-before.jsonl
    ```

3. Proceed with upgrading ODM instance

4. Run check over ODM instance after the upgrade

    ```shell
    docker run \
      --env ODM_URL=${ODM_URL} \
      --env ODM_TOKEN=${ODM_TOKEN} \
      091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/clickhouse-helper \
      odm sanity > sanity-result-after.jsonl
    ```

5. Run a diff:

    ```shell
    diff sanity-result-before.jsonl sanity-result-after.jsonl
    ```

Basically there should be no changes, except studies with already uploaded VCF files, they should be started to reimport.
