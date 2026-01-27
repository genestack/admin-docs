# Release notes

## Version 1.62

!!! tip ""
    Helm chart version 1.62.0

### Helm configuration changes

- Added a new service, `processors-controller`, which manages the lifecycle of `transformation` pods.
    - To use transformations, configure cross-account ECR access. See [Cross-account ECR access](./clouds/aws.md#cross-account-ecr-access).
    - The `processors-controller` requires RBAC permissions to manage `Pods`, `ConfigMaps`, and `PersistentVolumeClaims`.
      The required manifests are included in the Helm chart and can be disabled by setting `processorsController.rbac.enabled` to `false`.

- Updated clickhouse server and clickhouse keeper image declarations.
    - `altinity.clickhouse.installation.spec.templates.podTemplate.spec.mainContainer.image` and
      `altinity.clickhouseKeeper.installation.spec.templates.podTemplate.spec.mainContainer.image`
      were strings, but now they are objects containing fields for `registry`, `repository`, and `tag`.

      The same in YAML format.
      Old image declaration:

      ```yaml
      image: docker.io/clickhouse/clickhouse-keeper:25.8.4
      ```

      New image declaration:

      ```yaml
      image:
        registry: docker.io
        repository: clickhouse/clickhouse-keeper
        tag: 25.8.4
      ```

- Increased memory requirements for loading large cell expression datasets.
    - When loading datasets with millions of cells, increase memory limits for the `func-job` service to prevent out-of-memory errors.

      Example of configuration:

      ```yaml
      funcJob:
        resources:
          limits:
            memory: 5Gi
          requsts:
            memory: 5Gi
      ```
