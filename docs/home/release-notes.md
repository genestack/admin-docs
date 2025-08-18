# Release notes

## Version 1.62

!!! tip ""
    Helm chart version 1.62.0

### Helm configuration changes

- Added a new service, `processors-controller`, which manages the lifecycle of `transformation` pods.
    - To use transformations, configure cross-account ECR access. See [Cross-account ECR access](./clouds/aws.md#cross-account-ecr-access).
    - The `processors-controller` requires RBAC permissions to manage `Pods`, `ConfigMaps`, and `PersistentVolumeClaims`.
      The required manifests are included in the Helm chart and can be disabled by setting `processorsController.rbac.enabled` to `false`.
