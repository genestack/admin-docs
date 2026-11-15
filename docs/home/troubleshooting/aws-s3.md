# Problems with S3

## Storage service

The storage service is the only ODM component that reads and writes S3. It runs in one Pod with two containers,
`storage-service` and `rclone`. To read the storage service's log, replace `<namespace>` and `<release>`:

```shell
kubectl logs -n <namespace> deployment/<release>-odm-storage-service --all-containers
```

- S3 denies access to a file in the storage service's log. An import or download of a file S3 refuses to read
  answers 404. An upload S3 refuses to write answers 500. This indicates an issue with:

    - Access to S3.

    - Access to the KMS key (if the KMS key is used for encrypting data in the s3 bucket).

    - KMS key is not specified in the bucket's storage, under `genestack.storage.storages.s3` in the storage
      service's configuration (`storageService.secretFiles` in the Helm chart), if the KMS key is used for
      encrypting data in the s3 bucket. The `s3-bucket-sse-kms.yaml` file in the chart's `examples` directory
      shows the settings.

- Imports, uploads and downloads answer 503. The storage service, its rclone container or S3 is unavailable. Check
  that the storage service's Pod is ready, then read its log.

- An import answers 400 with a bucket name in the message. No storage under `genestack.storage.storages.s3` covers
  that bucket. Add a storage named after the bucket, or the storage `*`, which covers every bucket that has no
  storage of its own. Write its key as `"[*]"`: Spring drops a bare `*` from a map key. Then restart the storage
  service: it reads its storages at start. Files registered under a storage that the configuration no longer names
  also answer 400.
