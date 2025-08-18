# Release notes

## Version 1.56

### Export metrics to Genestack

Fluent-bit was introduced as an extra service tasked with collecting and dispatching metrics in Prometheus format to a Genestack.

These metrics encompass technical and/or product-related data, devoid of any sensitive information.

If you wish to deactivate this functionality, you can do so by configuring the following parameter:

```yaml
fluent-bit:
  enabled: false
```

### Helm configuration changes

Now organization name and hostname are in a `global` section:

From:

```yaml
odmFrontendHostname: odm.local
applications:
  configurationFiles:
    "application.yaml":
      frontend:
        ui:
          organization:
            name: "Genestack"
```

To:

```yaml
global:
  hostname: odm.local
  organizationName: "Genestack"
```
