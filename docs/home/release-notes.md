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
      From:

      ```yaml
      image: docker.io/clickhouse/clickhouse-keeper:25.8.4
      ```

      To:

      ```yaml
      image:
        registry: docker.io
        repository: clickhouse/clickhouse-keeper
        tag: 25.8.4
      ```
