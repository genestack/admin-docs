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

## Version 1.55

### Configure ODM usage together with encrypted S3 bucket (SSE-KMS and SSE-S3 only)

#### Introduction

!!! attention ""
    You can find configuration examples in the ODM Helm chart.

In case you have several AWS credentials in your configuration, you need to modify only the credentials for
accessing the bucket in specified as `frontend.aws.bucket`.

#### SSE-KMS

To enable uploading into an SSE-KMS encrypted bucket, you need to customize `func-file` configuration.
The following configuration example uses a bucket encrypted by SSE-KMS with the name `<BUCKET_NAME>`.
The bucket configuration should specify the algorithm `aws:kms` as `preferredAlgorithm`. Additionally,
the property `kmsCmkId` should be added with a value equal to key id `arn:aws:kms:...` if the bucket policy
requires this key to be explicitly send on PUT request. The `func-file` section in the configuration
should look like this:

#### SSE-S3

The SSE-S3 encryption type is default to the most buckets. To force ODM request
this type of encryption from S3 provider for `<BUCKET_NAME>`, you need to specify
the `preferredAlgorithm` property with the value `AES256`:

#### On `storage_config` section configuration in `func-file`

Keep in mind that `func-file` reads the `storage_config` section sequentially. You can create specific configurations
for individual buckets, e.g., if one has SSE-KMS encryption while others do not. To do this, as the first item in
the list, you'll need to specify the bucket with the specific configuration and its name. Then, provide the general
configuration for the other buckets using the wildcard symbol `*`. ODM will only upload files to the bucket, specified
as `frontend.aws.bucket` property, regardless to `storage_config` section.

### Genestack pod separation

Example on the image section, but it's applicable for sections with backend/frontend separation.

ApplicationSettings changes showed separately:

From:

```yaml
genestack:
  image:
    backend:
      registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
      repository: genestack/core
      pullPolicy: Always
      pullSecrets: []
    frontend:
      registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
      repository: genestack/applications
      pullPolicy: Always
      pullSecrets: []
```

To:

```yaml
core:
  image:
    registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
    repository: genestack/core
    pullPolicy: Always
    pullSecrets: []

applications:
  image:
    registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
    repository: genestack/applications
    pullPolicy: Always
    pullSecrets: []
```

### Application settings rework

From:

```yaml
genestack:
  applicationSettings:
    backend:
      properties:
        # backend.properties file content
      propertiesAuth:
        # backend-credentials.properties file content
      propertiesLimits:
        # limits.yaml file content
      predefinedSystemUsers:
        # token and password for technical odm users
      predefinedUsers:
        # predefined-users.json file content
    frontend:
      properties:
        # frontend.properties file content
        "google.openid.ini":
          # google.openid.ini file content
        "microsoft.openid.ini":
          # microsoft.openid.ini file content
        "okta.openid.ini":
          # okta.openid.ini file content
      propertiesAuth:
        # frontend-credentials.properties file content
      monitoringThresholds:
        # monitoring-thresholds.yaml file content
    saml:
      # saml directory content
```

To:

```yaml

core:
  configurationFiles:
    "application.yaml":
      # backend.properties and backend-credentials.properties files content in YAML format
    "settings.py.local":
      # settings.py.local file content
  secretFiles:
    # saml directory content

applications:
  configurationFiles:
    "application.yaml":
    # frontend.properties and frontend-credentials.properties files content in YAML format
    "google.openid.ini":
    # google.openid.ini file content
    "microsoft.openid.ini":
    # microsoft.openid.ini file content
    "okta.openid.ini":
    # okta.openid.ini file content
```

### High-level paths renaming in values.yaml

#### Solr

From:

```yaml
index: {}  # Solr configuration
```

To:

```yaml
solr: {}  # Solr configuration
```

#### Clickhouse

From:

```yaml
txIndex: {}  # Clickhouse configuration
```

To:

```yaml
clickhouse: {}  # Clickhouse configuration
```

#### Mysql

From:

```yaml
db: {}  # Mysql configuration
```

To:

```yaml
mysql: {}  # Mysql configuration
```

#### Nginx

From:

```yaml
proxy: {}  # Nginx configuration
```

To:

```yaml
nginx: {}  # Nginx configuration
```
