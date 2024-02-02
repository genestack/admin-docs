# External Services

- As a storage solution for the source data ODM supports [S3](https://en.wikipedia.org/wiki/Amazon_S3) and any Distribute File System that can be represented in Kubernetes as a StorageClass with RWM attribute. e.g. ([NFS](https://en.wikipedia.org/wiki/Network_File_System), [EFS](https://en.wikipedia.org/wiki/Amazon_Elastic_File_System))
- For notifications and secure tokens distribution SMTP service is required.
- For SSO (Single Sign-On) functionality an Identity Provider is required.
