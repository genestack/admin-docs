# How to deploy

## Requirements

- Kubernetes

- Helm 3+

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) for downloading Helm chart

## Deployment Information

- All main ODM settings are presented in the Deployment Helm Chart.

- To deploy ODM, you need a `StorageClass` and `IngressClass`.

- If you are using Prometheus Operator, then you can set up automatic collection of metrics through parameter
`serviceMonitor` in the Deployment Helm Chart.

- We provide a tool to automatically update credentials from the [ECR](https://aws.amazon.com/ecr/)
(in it we store our images), but you can use your own.

    - It should be remembered that the ECR password is given for 12 hours.

- If you are using an external data mount (for example NFS), then you must create a PVC in Kubernetes before deploying.

## Deployment process

1. [Configure AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html#getting-started-quickstart-new) (see **Long-term credentials**) with the credentials provided by our support team (this is necessary to access the ECR).

2. Login to ECR

    ```shell
    aws ecr get-login-password --region us-east-1 | helm registry login --username AWS \
    --password-stdin 091468197733.dkr.ecr.us-east-1.amazonaws.com
    ```

3. Download Helm Chart (use the latest version)

    ```shell
    helm pull oci://091468197733.dkr.ecr.us-east-1.amazonaws.com/genestack/chart/odm --version "1.56.0"
    ```

4. Unzip the archive and navigate to the directory.

5. Make changes to file `example-values/recommendations.yaml` according to your infrastructure and requirements.

6. Deploy

    ```shell
    helm upgrade -i odm --atomic --debug --timeout 4000s --create-namespace -n odm \
    -f example-values/recommendations.yaml ./
    ```
