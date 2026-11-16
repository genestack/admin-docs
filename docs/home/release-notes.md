# Release notes

## Version 1.66

!!! tip ""
    Helm chart version 1.66.0

### The storage service replaces the rclone Deployment

!!! danger

    ODM 1.66 refers to files by the storage service's ids, where earlier versions stored links. The storage service
    change itself does not migrate the files of instances installed before 1.66. The BIA-280 migration, which ships
    with 1.66, does. Until it has run, the calls that
    [`s3://` links into ODM's bucket work only for files the storage service recorded](#s3-links-into-odms-bucket-work-only-for-files-the-storage-service-recorded)
    lists fail for files 1.65 stored in ODM's bucket.

This release adds the storage service, the only ODM component that calls rclone. Core, applications and func-job
read, write and copy files through it. It runs two containers in one Pod, `storage-service` and `rclone`, under the `storageService` key
of the Helm chart. The standalone `rclone` Deployment and Service, and the `rclone` key, are removed.

The storage settings move to `storageService`:

| Before 1.66 | From 1.66 |
| --- | --- |
| `genestack.rclone` in the `core`, `applications` and `funcJob` files | `genestack.storage.storages` in `storageService.secretFiles` |
| `frontend.aws.bucket` and `frontend.aws.pathInBucket` in the `applications` files | `genestack.storage.internal.location` in `storageService.files`, a URL: `s3://<bucket>` or `s3://<bucket>/<pathInBucket>` |

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
            location: "s3://<AWS_S3_BUCKET_NAME>"
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

`genestack.storage.internal.location` is required. Build it from your old values: `s3://`, the bucket that
`frontend.aws.bucket` named, then `/` and `frontend.aws.pathInBucket` if you set one. For example,
`s3://<AWS_S3_BUCKET_NAME>` for a whole AWS bucket, or `s3://main` for a MinIO bucket named `main`. `s3://` is the
only scheme the storage service accepts. It does not start when the location is missing, malformed or of another
scheme, or when `genestack.storage.storages.s3` has neither an entry named after its bucket nor `"[*]"`.

The storage service keeps each file it writes as a path relative to the location. To move ODM's internal storage to
another bucket or prefix, copy its objects there, change the location and restart the storage service. The database
needs no change for the files the storage service recorded. Files stored by 1.65 and earlier are not recorded yet:
they join once BIA-280 migrates them.

### The storage service is a native image

The storage service's image, `genestack/storage`, runs a GraalVM native binary in place of a JVM. The image's
entrypoint passes it `-Xmx128m -XX:+ExitOnOutOfMemoryError`. Direct memory may grow to the heap's size, so the heap
and direct memory take at most 256 MiB together. The binary exits at its first heap `OutOfMemoryError`, as the JVM
services do, and Kubernetes restarts it.

Under ODM's functional tests the binary idles at 42 to 50 MiB, peaks at 121 MiB and starts in 0.17 s. The chart
requests 128Mi of memory for the `storage-service` container and limits it to 384Mi.

The native image differs from ODM's JVM services:

- `storageService.envVars` takes no `JAVA_OPTS`, `JAVA_ARGS`, `JAVA_HEAP_DEPENDS_ON_RAM`, `JAVA_DEBUG_*` or
  `JAVA_JMX_*`: the image has no `entrypoint.sh` to read them.
- `storageService.service` has no `debug` or `jmx` key, and the Pod opens no debug or JMX port.
- The actuator's metrics include no JVM garbage collection metrics.
- Spring fixes its conditional beans when the image is built. A setting such as `spring.flyway.enabled` in
  `storageService.files` has no effect.
- A failure at start prints plain text to the log, in place of one JSON line.

### Each file in ODM's bucket gets a folder of its own

The storage service picks the path of every file it writes into ODM's bucket, uploads and imported copies alike:
`<file id>/<file name>` under `genestack.storage.internal.location`. `<file id>` is the storage service's id of the
file, so no write replaces a file.

Before 1.66, ODM wrote attachments to `<study>/<file name>`, and UI imports and multipart signal, cell and metadata
imports to `<study>/<timestamp>/<file name>`. A UI attachment replaced a file of the same name. Files that 1.65
stored keep their paths.

When the study already has an attached file of the same name:

- An attachment import, `POST /api/v1/jobs/import/file` or `POST /api/v1/jobs/import/file/multipart`, answers 409
  "A file with this name is already attached to the study."
- A UI upload adds a version, or replaces the attached file when versioning is off.

Two attachment imports of one name into one study that run at the same time can both pass the 409 check. The second
then becomes a version of the first, or replaces it when versioning is off.

### `file://` links must lie under `/local_data`

A `file://` link must now lie under one of the directories in `genestack.storage.local-roots` of the storage
service's configuration, `/local_data` by default. The chart mounts `externalVolumes` at `/local_data`.

The storage service also refuses a link with a `..` segment, and a path that goes through or to a symbolic link,
even one that points inside a root. rclone 1.76 follows symbolic links whatever its options say. An import with a
refused `file://` link answers 400. Replace symbolic links on the volume with the files or directories they point to.

### `s3://` links into ODM's bucket work only for files the storage service recorded

The storage service refuses an `s3://` link inside `genestack.storage.internal.location` unless it recorded a file at
that path, such as a 1.66 upload. These fail on such a link:

- An import by link answers 400, and creates no job.
- An export of the file fails.
- A download task of the file fails.
- A metadata replace on an object that holds the link fails, and writes none of its values.

ODM sends a file link stored by 1.65 and earlier to the storage service when it first uses it. With the location
built from the 1.65 values, as above, every file 1.65 stored in ODM's bucket lies inside it, and the storage service
has no record of it until BIA-280 migrates it. ODM 1.66 relies on BIA-280's migration shipping with it.

`s3://` links to objects another tool put into ODM's bucket stay refused. To import one, copy it to another bucket
and import it from there.

For `s3://` links, the rule keeps one id for each object inside the location. Every such link names a file the
storage service wrote, which moves with the location, so a move needs no database change.

The rule does not cover `http(s)://` links. When the store serves an object in the location over HTTP without
credentials, as a public MinIO or S3 bucket does, the storage service records its `http(s)://` URL as an external
file, as it records any other HTTP link. That file keeps its URL after a move, and an object the storage service
wrote then has a second id.

### `http(s)://` links to loopback and link-local hosts are refused

The storage service refuses an `http(s)://` link whose host is `localhost`, or is or resolves to a loopback,
link-local, any-local or multicast address, such as `127.0.0.1`, `::1` or `169.254.169.254`. An import with such a
link answers 400. The storage service checks the host on every call, so a file whose host name later resolves to
such an address fails too. Serve such files from a host with a routable address.

### Storage errors answer distinct HTTP statuses

Imports, uploads, downloads, reference genome uploads and xrefset creation now answer the storage service's errors
as follows:

| Cause | HTTP status |
| --- | --- |
| The file does not exist, or the store refuses to read it | 404; imports add hint A0003 |
| The link is malformed, a `file://` link lies outside the local roots or goes through a symbolic link, an `http(s)://` link names a loopback or link-local host, or an `s3://` link points inside ODM's internal location at an object the storage service did not record | 400 |
| No storage in `genestack.storage.storages` covers the link's bucket or host, or the file's storage was removed from it | 400, naming the bucket or host |
| The name of an uploaded file, or the last segment of an attachment import's link, is empty, contains `/`, is `.` or `..`, has a control character, or exceeds 255 bytes | 400, before any job |
| The storage service, rclone or the store is unavailable | 503 |
| The store refuses a write, such as an upload to ODM's bucket, or any other error | 500 |

These answers changed:

| Request | Before 1.66 | From 1.66 |
| --- | --- | --- |
| An import with a link ODM cannot read | A job that fails with A0003 | 404 with hint A0003, and no job |
| An import with an `http(s)://` link to a loopback or link-local host | A job | 400, and no job |
| An import with an `s3://` link inside `genestack.storage.internal.location` to an object the storage service did not record | A job | 400, and no job |
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
