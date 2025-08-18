# Release notes

## Version 1.57

### Helm configuration changes

- Removed the link to the database for the service `func-file`. If you have it in your `values.yaml`, then you can safely remove the `spring` map completely.

```yaml
funcFile:
  configurationFiles:
    "application.yaml":
      spring:
        datasource:
          # -- Mysql jdbc URL
          url: "jdbc:mysql://..."
```

- For the Clickhouse `busybox` image, the ability to set the repository and version has been added.

```yaml
clickhouse:
  busyboxImage:
    # -- Image registry
    registry: docker.io
    # -- Image repository
    repository: busybox
    # -- Image tag
    tag: 1.36.1
```
