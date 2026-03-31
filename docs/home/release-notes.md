# Release notes

## Version 1.62

!!! tip ""
    Helm chart version 1.62.3

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

- Removed an irrelevant parameter `?appId=<OPENID_CLIENT_ID>` in OpenId settings (Azure only).

    Old configuration:

    ```yaml
    applications:
      files:
        "/var/lib/genestack/properties/microsoft.openid.ini": |
          ...
          discoveryDocumentUri=https://login.microsoftonline.com/<MICROSOFT_TENANT>/v2.0/.well-known/openid-configuration?appId=<OPENID_CLIENT_ID>
          ...
    ```

    New configuration:

    ```yaml
    applications:
      files:
        "/var/lib/genestack/properties/microsoft.openid.ini": |
          ...
          discoveryDocumentUri=https://login.microsoftonline.com/<MICROSOFT_TENANT>/v2.0/.well-known/openid-configuration
          ...
    ```

- Sensitive SMTP configuration parameters have been moved to a separate secret.

    Old configuration:

    ```yaml
    core:
      files:
        "/var/lib/genestack/properties/application.yaml":
          mail:
            smtp:
              host: "<SMTP_HOST>"
              port: "<SMTP_PORT>"
              security: "<SMTP_SECURITY>"
              username: "<SMTP_USER>"
              password: "<SMTP_PASSWORD>"
    ```

    New configuration:

    ```yaml
    core:
      files:
        "/var/lib/genestack/properties/application.yaml":
          mail:
            smtp:
              host: "<SMTP_HOST>"
              port: "<SMTP_PORT>"
              security: "<SMTP_SECURITY>"
      secretFiles:
        "/var/lib/genestack/properties/secret.yaml":
          mail:
            smtp:
              username: "<SMTP_USER>"
              password: "<SMTP_PASSWORD>"
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
