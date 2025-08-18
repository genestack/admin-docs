# Release notes

## Version 1.59

!!! tip ""
    Helm chart version 1.59.2

### Helm configuration changes

The following ODM components were removed: `clickhouse`, `mysql-prepare` and `clickhouse-helper`.

Removing the old `clickHouse` instance is completely safe, as its data was migrated to the cluster version of ClickHouse in the 1.58 release, and it should have been disabled after that.

### Mysql migration

Background migrations, running during the `core` container startup, may take up to tens of minutes. Use the Helm `--timeout 30m` option to adjust the timeout accordingly.

If the ODM service is unable to start within the allocated time, increase the `failureThreshold` value for the core container.

### Removal of Application Container Persistent Volume

We have removed the Persistent Volume Claim (PVC) and the necessary configuration for the `applications` container. This removal is safe, so there's no need for concern.

### Increasing memory limits for func-file

A mechanism of attaching files to studies has been extended with new data sources. Attached files are stored in an
S3 bucket, configured for `applications` container. The `func-file` container requires additional memory
to facilitate efficient data transfer. This is particularly critical for the parallel upload of large files
(several gigabytes in size) from the S3 source.

The default configuration works fine for sequential file uploads. To enable parallel uploads,
please increase the `limits` for memory, as well as the `cRTInitialReadBufferSizeInBytes` parameter. Additionally,
it is highly recommended that for JVM services, the values for the `requests` and `limits` parameters be equal.
Take a look at the `odm/examples/parallel-file-upload.yaml` file as an example of configuration,
demonstrating the settings required for the parallel upload of five files, each 10GB in size.
Please contact Genestack support if you need any further help with the configuration.

You can also check Amazon S3 limits here: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html>.

### Amazon S3 bucket configuration

The CORS policy is not necessary for the S3 bucket used by ODM. The policy could be removed from the bucket
configuration.
