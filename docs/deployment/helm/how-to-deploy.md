# How to deploy

## Requirements

- Kubernetes

- Helm 3+

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) for getting permissions to download Helm chart

## Deployment Information

- All main ODM settings are presented in the Deployment Helm Chart.

- To deploy ODM `StorageClass` and `IngressClass` have to be configured in Kubernetes Cluster.

- To enable automatic metrics collection when using Prometheus Operator, configure the `serviceMonitor` parameter in the Helm Chart.

- We provide a tool that automates the process of updating credentials for [ECR](https://aws.amazon.com/ecr/), where our container images are hosted.
This tool ensures seamless access to ECR by handling credential rotation.

    - It's important to note that the ECR password is valid for a duration of 12 hours.

- To utilize external storage solutions that are compatible with Distributed File System (DFS) within a Kubernetes environment, it is necessary to first establish a Persistent Volume Claim (PVC).

## Deployment process

1. [Configure AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html#getting-started-quickstart-new) (see **Long-term credentials**) with the credentials provided by our support team (this is necessary to access the ECR).

2. Login to ECR

    ```shell
    aws ecr get-login-password --region us-east-1 | helm registry login --username AWS \
    --password-stdin 091468197733.dkr.ecr.us-east-1.amazonaws.com
    ```

3. Download Helm Chart (use the latest version)

    ```shell
    helm pull oci://091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/chart/odm
    ```

4. Untar the archive.

    ```shell
    tar xvf odm-*.tgz
    ```

5. Take a look to a default `values.yaml` and to a `odm/example-values` directory, the most general file in there is `recomendations.yaml`, make changes according to your infrastructure and requirements in separate file, e.g. `custom-values.yaml`.

6. Deploy

    - In this example odm used as a release name, also as a namespace name

    ```shell
    helm upgrade -i odm --create-namespace -n odm -f custom-values.yaml ./
    ```
