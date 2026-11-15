# Release notes

## Version 1.66

!!! tip ""
    Helm chart version 1.66.0

### The storage service replaces the rclone Deployment

!!! danger

    ODM 1.66 refers to files by the storage service's ids, where earlier versions stored links. This release does
    not migrate the files of instances installed before 1.66.

This release adds the storage service, the only ODM component that calls rclone. Core, applications and func-job
read, write and copy files through it. It runs two containers in one Pod, `storage-service` and `rclone`, under the `storageService` key
of the Helm chart. The standalone `rclone` Deployment and Service, and the `rclone` key, are removed.

The storage settings move to `storageService`:

| Before 1.66 | From 1.66 |
| --- | --- |
| `genestack.rclone` in the `core`, `applications` and `funcJob` files | `genestack.storage.storages` in `storageService.secretFiles` |
| `frontend.aws.bucket` and `frontend.aws.pathInBucket` in the `applications` files | `genestack.storage.internal.bucket` and `genestack.storage.internal.pathInBucket` in `storageService.files` |

Remove every `genestack.rclone` block and both `frontend.aws` settings from your values. `storageService.secretFiles`
is stored in a Kubernetes Secret, so the storages' credentials stay out of ConfigMaps. The `s3-bucket-*.yaml` files
in the chart's `examples` directory show the new shape. For example:

```yaml
storageService:
  files:
    "/app/config/application.yaml":
      genestack:
        storage:
          internal:
            bucket: "<AWS_S3_BUCKET_NAME>"
            pathInBucket: ""
  secretFiles:
    "/app/config/storages/application.yaml":
      genestack:
        storage:
          storages:
            s3:
              "[*]":
                provider: "AWS"
                envAuth: true
                region: "<AWS_REGION>"
            http:
              noEscape: true
            local: {}
```

### `file://` links must lie under `/local_data`

A `file://` link must now lie under one of the directories in `genestack.storage.local-roots` of the storage
service's configuration, `/local_data` by default. The chart mounts `externalVolumes` at `/local_data`.

The storage service also refuses a link with a `..` segment, and a path that goes through or to a symbolic link,
even one that points inside a root. rclone 1.76 follows symbolic links whatever its options say. An import with a
refused `file://` link answers 400. Replace symbolic links on the volume with the files or directories they point to.

Before 1.66, a `file://` link could read any file in the rclone container, the Pod's service account token included.

### Storage errors answer distinct HTTP statuses

Imports, uploads, downloads, reference genome uploads and xrefset creation now answer the storage service's errors
as follows:

| Cause | HTTP status |
| --- | --- |
| The file does not exist, or ODM has no access to it | 404; imports add hint A0003 |
| The link is malformed, or a `file://` link lies outside the local roots or goes through a symbolic link | 400 |
| No storage in `genestack.storage.storages` covers the link's bucket or host, or the file's storage was removed from it | 400, naming the bucket or host |
| The destination path of an upload or a copy already holds a file | 409 |
| The storage service, rclone or the store is unavailable | 503 |
| Any other error | 500 |

These answers changed:

| Request | Before 1.66 | From 1.66 |
| --- | --- | --- |
| An import with a link ODM cannot read | A job that fails with A0003 | 404 with hint A0003, and no job |
| `POST /api/v1/reference-genomes` with an annotation link ODM cannot read | 400 | 404 |
| `POST /api/v1/reference-genomes` with an annotation source ODM cannot reach | 400 | 503 |
| `POST /api/v1/xrefsets` with a link ODM cannot read | 400 | 404 |

`ftp://` annotation links of reference genomes answer as before.

### Search no longer matches file links

ODM 1.66 stores a storage service file id where it stored a link, in three metadata keys: `genestack.url:data`,
`genestack:metadata$Source` and `genestack.url:annotations`. Solr and facets skip these ids. So free-text search no
longer matches text in these links. API responses still show the links.

This includes reference genome search, the `query` parameter of `GET /api/v1/reference-genomes`. Before 1.66, a
query such as `<bucket>/<dir>` found genomes by their annotation link.

## Version 1.65

!!! tip ""
    Helm chart version 1.65.0

## Version 1.64

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.64.0

### Genestack audit logs migration from MySQL to Clickhouse

This release migrates MySQL `genestack_audit` schema to ClickHouse `audit.Event` table and
removes `genestack_audit` schema from MySQL upon successful completion. Migration component responsible for it
is planned for a removal in the next release. As a result, this ODM version cannot be skipped during upgrades.

### Excluding audit events from tracking

It is now possible to exclude specific audit events from tracking if they are not needed, which saves disk space.
For example, you can exclude events related to data reads.

A configuration example is available in the `examples` directory of the Helm chart.

### Audit log retention period

!!! danger

    The retention period cannot be changed after installation. Set it before you install or upgrade to this version.

ODM keeps audit events in ClickHouse for a limited number of months and deletes older events automatically.
The default retention period is 12 months. To use a different one, set it in the Helm chart values:

```yaml
core:
  files:
    "/var/lib/genestack/properties/application.yaml":
      backend:
        clickhouse:
          audit:
            retentionMonths: 24
```
