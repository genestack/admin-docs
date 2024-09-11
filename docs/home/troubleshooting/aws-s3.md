# Problems with S3

## Func-file service

- `Can't (re-)connect to: https://bucket-name.s3.eu-west-1.amazonaws.com/file-name ... Access Denied` - This error indicates that there is an issue with:

  - Access to S3.

  - Access to the KMS key (if the KMS key is used for encrypting data in the s3 bucket).

  - KMS key is not specified in the ODM configuration `func-file` (if the KMS key is used for encrypting data in the s3 bucket).
