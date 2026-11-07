# Problems with S3

## Rclone service

ODM reaches S3 through the `rclone` service: imports read their files through it, and it copies attachments into
ODM's bucket. Core and applications give rclone the S3 settings, as the remotes under `genestack.rclone.storage.s3`
in the `application.yaml` of `core.files` and of `applications.files`. The two `genestack.rclone` sections must be
identical, or the chart does not render. The `s3-bucket-*.yaml` files in the chart's `examples` directory show these
settings for an IAM role, cross-account access, SSE-KMS and MinIO.

- An import from `s3://…` fails with "The system cannot open the file …" in the REST API, or "The system couldn't
  open the file …" in the web interface - rclone could not read the file. Its log says why. The log is JSON, and its
  errors have `"level":"error"`. With the release and namespace both named `odm`, as in
  [How to deploy](../helm/how-to-deploy.md):

    ```shell
    kubectl -n odm logs deployment/odm-rclone | grep '"level":"error"'
    ```

    The same error appears when applications cannot reach rclone at all. Then rclone's log shows nothing for it:
    check that the `odm-rclone` pod is running and ready.

- If rclone's errors say that access is denied, there is an issue with:

    - Access to S3: the remote's credentials, `accessKeyId` and `secretAccessKey`, or the role of ODM's service
      account with `envAuth: true`, cannot reach the bucket.

    - Access to the KMS key (if the KMS key is used for encrypting data in the s3 bucket).

    - The KMS key is not specified in the remote (if the KMS key is used for encrypting data in the s3 bucket): set
      `serverSideEncryption: "aws:kms"` and `sseKmsKeyId`, as in `examples/s3-bucket-sse-kms.yaml`.
