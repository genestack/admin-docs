# Release notes

## Version 1.58

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.58.3

### Clickhouse migration

In current release we moved from standalone Clickhouse container to the Clickhouse cluster that will be controlled by [Altinity clickhouse operator](https://github.com/Altinity/clickhouse-operator).  
We automated process of transferring data from the standalone version to the cluster version of clickhouse, migration will be executed during upgrade process.

Things that you have keep in mind before upgrading process:

- Migration time depends on resources(mostly on CPU and Disk IO) allocated for Clickhouse instances, during our tests we've mentioned average speed 50Gb per hour for instances with 4CPU/16Gb RAM

- It's necessary **not to set** flags as `--wait` and `--timeout` during upgrading process because of migration time

- The new ClickHouse cluster must have 25% more disk space than the standalone variant.

- Optional: We've developed [a tool that checks consistency of data](../troubleshooting/sanity-check.md), you could use it before and after the upgrading process and compare results of those checks just to be sure that everything went as expected

#### Following steps

  1. (Upgrade flow) Proceed with the [odm](../helm/how-to-deploy.md#deployment-process) installation, take itnto account:

     a. `odm-ops` chart will install the Altinity ClickHouse operator with pre-configured settings.

     b. In your custom values for `odm` chart adjust parameters for ClickHouse and Altinity ClickHouse (requests, limits, disk size +25%, etc.). We suggest to temporary increase resources for both of Clickhouses since it will reduce timing for migration.

     c. VERY IMPORTANT! Make sure that old clickhouse is NOT disabled! Path in values is `clickhouse.enabled`. By default, it's enabled.

     d. DO NOT apply `recommendations.yaml` file from example as is, it's recommendation ONLY for new installations!

     e. Resources path in values for old clickhouse is `clickhouse.resources` for the new one `altinity.clickhouse.installation.spec.templates.podTemplate.spec.mainContainer.resources`

     f. Persistance size path for old clickhouse is `clickhouse.persistence.size` for the new one `altinity.clickhouse.installation.spec.templates.volumeClaimTemplate.spec.resources.requests.storage`

  2. A job named `odm-clickhouse-helper` will appear in Kubernetes, and it will handle the migration.

     a. During the ClickHouse migration, ODM will continue to operate, but all writes to ClickHouse will be queued.  

     b. Wait until the `odm-clickhouse-helper` job completes, indicating that the migration is done.
  
  3. Disable `clickhouse` and `clickhouseHelper` in Helm values. You can refer to the example `disable-old-clickhouse-after-upgrade.yaml`.

  4. Update ODM one last time with `helm upgrade ...`. This will disable the old ClickHouse.

### Helm examples changes

- New examples for different ODM configuration options have been added to the `examples` helm chart directory, and all old ones have been updated.

- Additionally, recommendations for computing resources have been included.

### Helm configuration changes

- From this release, we are using fully original Docker images for the OSS components of ODM. It is not recommended to update them independently.

    From:

    ```yaml
    mysql:
      image:
        registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
        repository: genestack/mysql

    mailcatcher:
      image:
        registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
        repository: genestack/mailcatcher

    clickhouse:
      image:
        registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
        repository: genestack/clickhouse

    nginx:
      image:
        registry: 091468197733.dkr.ecr.us-east-1.amazonaws.com
        repository: genestack/nginx
    ```

    To:

    ```yaml
    mysql:
      image:
        registry: docker.io
        repository: mysql

    mailcatcher:
      image:
        registry: docker.io
        repository: dockage/mailcatcher

    clickhouse:
      image:
        registry: docker.io
        repository: clickhouse/clickhouse-server

    nginx:
      image:
        registry: docker.io
        repository: nginxinc/nginx-unprivileged
    ```

- Now you can mount any file with any content into any container in ODM! For example, your certificates. This feature required adding the full path in all existing ODM configuration files.

    From:

    ```yaml
    core:
      configurationFiles:
        "application.yaml":

    applications:
      configurationFiles:
        "application.yaml":
        "microsoft.openid.ini":
        "okta.openid.ini":
        "google.openid.ini":

    mysql:
      configurationFiles:
        "genestack.cnf":

    funcFile:
      configurationFiles:
        "application.yaml":

    funcJob:
      configurationFiles:
        "application.yaml":

    linkService:
      configurationFiles:
        "application.yaml":

    clickhouse:
      configurationFiles:
        "config.yaml":
        "users.yaml":

    nginx:
      configurationFiles:
        "odm.conf":
        "proxy-pass-parameters.conf":
    ```

    To:

    ```yaml
    core:
      files:
        "/var/lib/genestack/properties/application.yaml":

    applications:
      files:
        "/var/lib/genestack/properties/application.yaml":
        "/var/lib/genestack/properties/microsoft.openid.ini":
        "/var/lib/genestack/properties/okta.openid.ini":
        "/var/lib/genestack/properties/google.openid.ini":

    mysql:
      files:
        "/etc/mysql/conf.d/genestack.cnf":

    funcFile:
      files:
        "/app/config/application.yaml":

    funcJob:
      files:
        "/app/config/application.yaml":

    linkService:
      files:
        "/app/config/application.yaml":

    clickhouse:
      files:
        "/etc/clickhouse-server/config.d/config.yaml":
        "/etc/clickhouse-server/users.d/users.yaml":

    nginx:
      files:
        "/etc/nginx/conf.d/odm.conf":
        "/etc/nginx/conf.d/proxy-pass-parameters.conf":
    ```

- The AWS credentials for connecting to S3 in `core` and `applications` have been removed. If you have these parameters, you can safely delete them.

    ```yaml
    core:
      files:
        "/var/lib/genestack/properties/application.yaml":
          backend:
            aws:
              region: ""
              endpoint:
                url: ""
              access:
                key: ""
              secret:
                key: ""
    ```

    !!! danger

        Important! The AWS region in the `application` must remain! You can delete only the `endpoint`, `access` and `secret` parameters.

    ```yaml
    applications:
      files:
        "/var/lib/genestack/properties/application.yaml":
          frontend:
            aws:
              region: "{{ .Values.credentials.awsS3Region }}"
              endpoint:
                url: ""
              access:
                key: ""
              secret:
                key: ""
    ```

- Configuration file `settings.py.local` has been removed. If you are using it, you can safely delete it.

    ```yaml
    core:
      files:
        "settings.py.local":
    ```

- The previously added `BusyBox` image for `ClickHouse` has been removed. If you are using it, you can safely delete it.

    ```yaml
    clickhouse:
      busyboxImage:
        registry: docker.io
        repository: busybox
        tag: 1.36.1
    ```
