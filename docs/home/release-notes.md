# Release notes

## Version 1.66

!!! tip ""
    Helm chart version 1.66.0

### The storage service replaces the rclone Deployment

!!! danger

    ODM 1.66 refers to files by the storage service's ids, where earlier versions stored links. After the upgrade,
    core moves the links of earlier versions to the storage service in the background, which may take hours on a
    large instance. Core reports ready at once. Until it has tried each link, core starts no task and skips its
    garbage collection. Meanwhile, a file whose link has not moved yet may fail to read. Before you upgrade, make
    the storage service's storages cover every bucket the old links name. A file whose link fails to move stays
    unreadable until a later start of core moves it. See
    [The upgrade moves file links to the storage service](#the-upgrade-moves-file-links-to-the-storage-service).

This release adds the storage service, the only ODM component that calls rclone. Core, applications and func-job
read, write and copy files through it. It runs two containers in one Pod, `storage-service` and `rclone`, under the `storage` key
of the Helm chart. The standalone `rclone` Deployment and Service, and the `rclone` key, are removed.

The storage settings move to the `storage` key:

| Before 1.66 | From 1.66 |
| --- | --- |
| `genestack.rclone` in the `core`, `applications` and `funcJob` files | `genestack.storage.storages` in `storage.secretFiles` |
| `frontend.aws.bucket` and `frontend.aws.pathInBucket` in the `applications` files | `genestack.storage.internal.location` in `storage.files`, a URL: `s3://<bucket>` or `s3://<bucket>/<pathInBucket>` |

Remove every `genestack.rclone` block and both `frontend.aws` settings from your values. `storage.secretFiles`
is stored in a Kubernetes Secret, so the storages' credentials stay out of ConfigMaps. The `s3-bucket-*.yaml` files
in the chart's `examples` directory show the new shape. For example:

```yaml
storage:
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
needs no change for the files the storage service recorded. The upgrade records the files that 1.65 and earlier
stored inside the location the same way, so they move with it.

### The upgrade moves file links to the storage service

After every start, core moves the file links that ODM 1.65 and earlier stored to the storage service. It runs the
migration on a thread of its own and reports ready at once, so the upgrade's rollout does not wait for it. On a large
instance the migration may take hours. A fresh installation has no such links, and its scan finds none.

First, core scans table `genestack.SimpleMetainfo` in batches of 10,000 objectIds. It lists the links it finds in
table `genestack.FileLinkMigration`, one row per distinct value:

- the links in `genestack.url:data`, `genestack:metadata$Source` and `genestack.url:annotations`;
- the `s3://` URLs in `genestack.location:*`.

Template attributes keep their links. Each batch is one transaction, and table `genestack.FileLinkMigrationScan`
keeps the objectId the scan reached. A restart of core resumes the scan there. Once the scan ends, no later start
scans again.

Then core registers each row's link with the storage service:

- An `s3://` link inside `genestack.storage.internal.location` becomes an internal file at its own path. Core's
  garbage collector deletes it as it deletes a 1.66 upload: with the object that holds it, when that object is
  marked `File copy stored`.
- Any other link becomes an external file, as an import by link makes it.
- A link whose file is gone or unreadable becomes a file too, with availability `MISSING` or `UNREACHABLE`, so its
  bucket and key stay recorded.
- A `raw:`, `ftp:`, `storage:` or `localfile:` link stays as it is, and its row goes.

Core then replaces the link with the file's id, and re-indexes in Solr the objects that held it and their tasks.
Before the first link, core waits for the storage service as long as it takes. A call that finds the storage service
unavailable is retried 3 times, then fails its link. When the migration stops on another error, such as a lost
database connection, core logs `Moving the file links stored before ODM 1.66 failed, starting again in 60 s`. A
new run of the migration starts 60 s later, where the last one stopped.

Core logs the migration's progress:

- `Core holds its queued tasks and its garbage collector until the file links stored before ODM 1.66 are moved:
  GET /frontend/endpoint/actuator/filelinkmigration shows the progress`, at start, when core holds them;
- `File link scan: objectId <n> of <max>, <n> links found in <n> values`, at most every 30 s while it scans;
- `Waiting for the storage service to move the file links stored before ODM 1.66, for <n> s`, every minute while
  the storage service does not answer;
- `File links: <done> of <total> rows done`, after every 500 rows;
- `The file links stored before ODM 1.66 are moved: core starts its queued tasks and its garbage collector`, when
  the migration is done.

Until the migration is done:

- Core starts no task. The tasks queued at the upgrade, and the tasks users start meanwhile, such as initialization,
  export and download tasks, stay queued. Core starts them when the migration is done.
- Core's garbage collector skips its passes, so the files of removed objects stay in their storage until then.
- A file whose link has not moved yet may fail to read. An attached file that holds a 1.65 link fails to load. An
  import by an `s3://` link to a file that 1.65 stored in ODM's bucket answers 400. Creating an initialization task
  for such a file can fail, since core reads the file to create the task.

The migration is done when the scan has ended and core has tried each row once. A link that failed does not keep it
from being done. At a later start, core holds no task and no garbage collection when the scan has ended and every
row left failed at an earlier start. It retries those rows in the background.

`GET /frontend/endpoint/actuator/filelinkmigration` shows the migration's progress. Like the health check, it needs
no login. It shows counts, and in `error` why the migration is not done yet. A link that fails goes to its row of
`genestack.FileLinkMigration`, never to `error`. Applications relays core's answer, and answers 503 with an `error`
field when core does not answer.

| Field | Value |
| --- | --- |
| `state` | `NOT_STARTED` until the migration starts, `SCANNING`, `MIGRATING`, then `DONE`. A start after the scan ended skips `SCANNING` |
| `scannedObjectId` | The objectId the scan reached |
| `maxObjectId` | The largest objectId the scan reads to |
| `rows` | The rows of `genestack.FileLinkMigration` this run moves, found by the scan or left by an earlier run. 0 while core scans |
| `migrated` | The rows moved to the storage service in this run, those whose file is `MISSING` or `UNREACHABLE` included |
| `failed` | The rows that failed in this run and stay in `genestack.FileLinkMigration`, such as a link no storage covers. A link whose file is gone or unreadable counts in `migrated`, except a link inside `genestack.storage.internal.location` whose read is refused or fails |
| `kept` | The rows whose link stays as it is, such as a `raw:` link, in this run |
| `startedAt`, `finishedAt` | When this run started, and when it was done. Absent until then |
| `error` | `waiting for the storage service since <instant>` while core waits for it, with an ISO-8601 UTC time such as `2026-11-17T10:00:00.123Z`. Otherwise the failure of the last run that failed since core started, as `<class>: <message>`, such as `SQLException: Lock wait timeout exceeded`. Absent when there is neither, and once `DONE` |

The counts are per run. A run that fails starts again 60 s later, and counts from 0. `error` keeps its failure until
a run is `DONE`.

A link whose file is gone or unreadable moves to the storage service, which registers the file with an availability:

- `MISSING` when the store answers that there is no such file, such as a 404, or DNS knows no host of an `http(s)://`
  link. An `s3://` link inside `genestack.storage.internal.location` whose object is gone becomes an internal file
  with availability `MISSING`.
- `UNREACHABLE` when the store refuses the read, with 403 or 401, or fails on the way, such as a timeout, a host
  that gives rclone no answer within 60 s, or throttling. Inside `genestack.storage.internal.location` such a link
  fails instead: the storage service reads ODM's bucket with its own credentials.

Such a file has size -1 and no other metadata, and the storage service logs one WARN line for it. It stays
unreadable until the store gives it. The storage service's scheduled availability check reads each external file
again, the least recently checked first. Two chart keys set it:

- `storageService.files."/app/config/application.yaml".genestack.storage.availabilityCheck.cron`: when the check
  runs, a Spring cron expression. Default `0 0 2 * * *`, daily at 02:00.
- `storageService.files."/app/config/application.yaml".genestack.storage.availabilityCheck.batchSize`: how many
  files one run reads. Default `1000`.

The first check that finds the file records its metadata, and the file becomes `AVAILABLE`. The check reads no
internal file, so an internal file stays `MISSING`.

A link fails when no storage covers its bucket or host, the storage service refuses it, such as a `file://` link
outside `/local_data`, or rclone is down or answers another error. A `genestack.location:*` value that holds
several URLs, such as a FASTA file and its quality file, fails too: a storage file names one file. A failed link
keeps its old value, and its row stays in the table with `error` and `attempts`. Core logs one WARN line per failed
link, then a summary that starts `File links stored before ODM 1.66: <n> migrated, <n> failed, <n> kept as they are`.
An attached file whose link failed stays unreadable until its link moves. Every later start of core retries the rows
left.

Before the upgrade, make `genestack.storage.storages` cover every bucket the old links name, with credentials that
can read it. An `"[*]"` entry covers every bucket that has no entry of its own. Old `file://` links must lie under
`genestack.storage.local-roots`.

After the upgrade:

1. Watch the migration until `state` is `DONE`. `error` shows what it waits for or what failed last:

    ```shell
    curl https://<odm-host>/frontend/endpoint/actuator/filelinkmigration
    ```

2. List the links that failed. Once `state` is `DONE`, the rows left in the table are those links:

    ```sql
    SELECT value, error, attempts, objects FROM genestack.FileLinkMigration;
    ```

    No rows means every link moved. `error` holds the storage service's status and message, such as
    `FAILED_PRECONDITION No storage covers bucket <bucket>: genestack.storage.storages names none for it`. `objects`
    counts the values that held the link when the scan found it.

3. Fix the cause, such as a missing storage, then restart core:

    ```shell
    kubectl rollout restart -n <namespace> deployment/<release>-odm-core
    ```

    Core retries the rows left in the background, and holds no task meanwhile.

4. List the files that moved without their file, in the storage service's database, `storage`. External files:

    ```sql
    SELECT link, availability, checked_at FROM external_file WHERE availability IN ('MISSING', 'UNREACHABLE');
    ```

    Internal files, the links into ODM's bucket whose object was gone:

    ```sql
    SELECT path FROM internal_file WHERE availability = 'MISSING';
    ```

    `path` is relative to `genestack.storage.internal.location`. The first query also lists external files that a
    scheduled check found missing or unreachable since. When a storage's credentials made files `UNREACHABLE`, fix
    them in `genestack.storage.storages` and restart the storage service. The next check that finds such a file
    marks it `AVAILABLE`.

The upgrade leaves the parameters of func-job's jobs as they are. Func-job marks every job running at the upgrade
FAILED, as at any start. `PUT /api/v1/jobs/{jobExecId}/restart` of a job started before 1.66 answers 409 with hint
A0010, `Job <id> started before ODM 1.66 and holds file links: start a new import`. Start a new import instead.

### The storage service is a native image

The storage service's image, `genestack/storage`, runs a GraalVM native binary in place of a JVM. The image's
entrypoint passes it `-Xmx128m -XX:+ExitOnOutOfMemoryError`. Direct memory may grow to the heap's size, so the heap
and direct memory take at most 256 MiB together. The binary exits at its first heap `OutOfMemoryError`, as the JVM
services do, and Kubernetes restarts it.

Under ODM's functional tests the binary idles at 42 to 50 MiB, peaks at 121 MiB and starts in 0.17 s. The chart
requests 128Mi of memory for the `storage-service` container and limits it to 384Mi.

The native image differs from ODM's JVM services:

- `storage.envVars` takes no `JAVA_OPTS`, `JAVA_ARGS`, `JAVA_HEAP_DEPENDS_ON_RAM`, `JAVA_DEBUG_*` or
  `JAVA_JMX_*`: the image has no `entrypoint.sh` to read them.
- `storage.service` has no `debug` or `jmx` key, and the Pod opens no debug or JMX port.
- The actuator's metrics include no JVM garbage collection metrics.
- Spring fixes its conditional beans when the image is built. A setting such as `spring.flyway.enabled` in
  `storage.files` has no effect.
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
that path, such as a 1.66 upload or a file the upgrade moved. These fail on such a link:

- An import by link answers 400, and creates no job.
- An export of the file fails.
- A download task of the file fails.
- A metadata replace on an object that holds the link fails, and writes none of its values.

With the location built from the 1.65 values, as above, every file 1.65 stored in ODM's bucket lies inside it. The
upgrade records each of them at its own path, as
[The upgrade moves file links to the storage service](#the-upgrade-moves-file-links-to-the-storage-service) says,
so they keep working once their links have moved. Until then, an import by link and a metadata replace fail on them.
Core holds export and download tasks until the migration is done. A file whose link fails to move stays
unrecorded, and these calls fail on it until a later start of core moves it.

`s3://` links to objects another tool put into ODM's bucket stay refused. To import one, copy it to another bucket
and import it from there.

For `s3://` links, the rule keeps one id for each object inside the location. Every such link names a file the
storage service recorded, which moves with the location, so a move needs no database change.

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
