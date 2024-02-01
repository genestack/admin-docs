# Amazon WEB Services (AWS)

## Required types of resources

- [AWS S3 bucket](https://aws.amazon.com/s3/)
- [AWS IAM](https://aws.amazon.com/iam/) user with bucket access above (programmatic access)
- [AWS EKS](https://aws.amazon.com/eks/)
    - [AWS EBS](https://aws.amazon.com/ebs/)
    - [AWS ALB](https://aws.amazon.com/elasticloadbalancing/)
    - [AWS ACM](https://aws.amazon.com/certificate-manager/)
    - [AWS Route53](https://aws.amazon.com/route53/)

## Deployment

You can use any convenient way for you to create resources in AWS.

- CloudFormation
- Terraform
- Pulumi
- Manual creation

## Recommended steps for creating AWS resources

1. Create [S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html).

    - S3 bucket can be private.

    - You can use the S3 bucket only from one region, it is recommended to use a common region with EKS.

    - The S3 bucket can use KMS or default S3 encryption.

    - You need to configure CORS, you can find an example here [CORS configuration for S3](#configuration-examples)

2. Create IAM [user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html), [policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html) (it will be listed below) and [attach](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html) the policy to the user.

    - [Programmatic access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) is required to work ODM with IAM user.

    - You can find the recommended IAM policy here [IAM policy for S3 access](#configuration-examples).

3. Create [VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html).

    - Must have at least 2 private subnets [[link](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html)].

    - Each subnets must contain at least 100 free IP addresses [recommendation].

4. Create [EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html).

    - Deploy the addons you need.

5. Create Route53 [hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html).

    - You can use private or public zone.

    - It is recommended to use `A` type of record and a simple routing method.

    - The recording name should refer to your service (for example ALB).

6. Create [ACM](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) with your FQDN.

    - You can use wildcard certificate.

7. Create [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html) with ACM above.

    - If you do not have strict security requirements for ALB, then it is better to use the automatic creation of ALB using the ALB controller (details below in [EKS configuration](#recommended-steps-for-eks-configuration)).

    - You can use either internal or public schema depending on your situation (with an internal scheme, SCIM synchronization with Azure AD will not work, only SSO Authorization).

    - ALB Configuration

        - HTTP/2 - `On`

        - Idle timeout - `4000 seconds`

## Recommended steps for EKS configuration

1. Deploy [EBS controller](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

    - It is recommended to use GP3 storages (to do this, you need to deploy a specific [StorageClass](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/862fe33d2174e00f449ac34859c9990340234a63/docs/parameters.md)), but if required, you can use the default GP2 as well.

2. Deploy [ALB controller](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html).

    - If you need to use already created ALB, follow [this documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/use_cases/self_managed_lb/).

        1. Create and deploy [TargetGroupBinding](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/use_cases/self_managed_lb/#create-targetgroupbinding-crd) ([example](#configuration-examples)) in Kubernetes.

        2. [Add](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html) target group to ALB listener rules.

            - HTTP Path Pattern is `/*`.

            - HTTP Host Header is `ODM_FQDN`. `ODM_FQDN` - ODM url without `https://` or `http://`. For example: `odm.genestack.com`.

        3. Forward to target group above

        4. Set `nginx.ingress.enabled` parameter to `false` in ODM helm chart.

        5. Make sure that health checks pass, you may need to make changes to the Security Groups of the ALB (after ODM deployment).

    - If you don't have an ALB.

        - It will be created automatically during ODM deployment.

3. [Optional] Deploy [External DNS](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md).

    - To automatically create records in Route53, it is recommended to use External DNS.

    - Be careful when using it, you should limit the list of zones with which it can work. And if the zone is used for something else, then make sure that it will not delete records from there.

## Configuration examples

<details><summary>IAM policy for S3 access</summary>
```json
{
  "Version" : "2012-10-17",
  "Statement" : [
    {
      "Sid" : "ListObjectsInBucket",
      "Effect" : "Allow",
      "Action" : [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Resource" : [S3_BUCKET_ARN]
    },
    {
      "Sid" : "AllObjectActions",
      "Effect" : "Allow",
      "Action" : [
        "s3:*Object*",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource" : [S3_BUCKET_ARN/*]
    },
    {
      "Sid" : "AllowUseOfTheKey",
      "Effect" : "Allow",
      "Action" : [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource" : KMS_KEY_ARN
    }
  ]
}
```
</details>

<details><summary>CORS configuration for S3</summary>
```json
[
    {
        "AllowedHeaders": [
            "accept",
            "accept-language",
            "Content-Type"
        ],
        "AllowedMethods": [
            "PUT"
        ],
        "AllowedOrigins": [
            "https://ODM_FQDN"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    }
]
```
</details>

<details><summary>TargetGroupBinding example</summary>
```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: odm
  namespace: odm
spec:
  ipAddressType: ipv4
  serviceRef:
    name: odm-nginx
    port: 80
  targetGroupARN: arn:aws:elasticloadbalancing:us-east-1:12345678902:targetgroup/odm/d74e4eed524cddcd
```
</details>
