# Amazon Web Services (AWS)

## Required types of resources

- [AWS S3 Bucket](https://aws.amazon.com/s3/)
- [AWS IAM](https://aws.amazon.com/iam/) user with bucket access above (programmatic access) [_up to ODM version 1.60_]
- [AWS IAM](https://aws.amazon.com/iam/) role with bucket access above [_after ODM version 1.60_]
- [AWS EKS](https://aws.amazon.com/eks/)
    - [AWS EBS](https://aws.amazon.com/ebs/)
    - [AWS ALB](https://aws.amazon.com/elasticloadbalancing/)
    - [AWS ACM](https://aws.amazon.com/certificate-manager/)
    - [AWS Route53](https://aws.amazon.com/route53/)

## Steps for creating AWS resources

1. Create [S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html).

    - You can use the S3 Bucket only from one region, it is recommended to use a common region with EKS.

    - The S3 Bucket can use KMS or default S3 encryption.

    - It is recommended to set up a lifecycle rule to clean up unfinished multipart uploads. Example of [AbortIncompleteMultipartUpload rule](#configuration-examples).

2. Create [VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html).

    - Must have at least 2 private subnets [[link](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html)].

    - Each subnets must contain at least 100 free IP addresses [recommendation].

3. Create [EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html).

    - Deploy the addons you need.

4. Create IAM [user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) [_up to ODM version 1.60_], [role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html) [_after ODM version 1.60_], [policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html) (it will be listed below) and [attach](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html) the policy to the user/role.

    - [Programmatic access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) is required to work ODM with IAM user.

    - You can find the recommended IAM policy here [IAM policy for S3 access](#configuration-examples).

    - Choose between [EKS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) and [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) based on your requirements; however, we recommend using EKS Pod Identity.

    - You can find the recommended `Trust Relationships` for EKS IRSA here [IRSA Trust Relationships](#configuration-examples).

    - You can find the recommended `Trust Relationships` for EKS Pod Identity here [Pod Identity Trust Relationships](#configuration-examples).

5. Create Route53 [hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html).

    - You can use private or public zone.

    - It is recommended to use `A` type of record and a simple routing method.

    - The recording name should refer to your service (for example ALB).

6. Create [ACM](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) with your FQDN.

    - You can use wildcard certificate.

7. Create [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html) with ACM above.

    - If you do not have strict security requirements for ALB, then it is better to use the automatic creation of ALB using the ALB controller (details below in [EKS configuration](#recommended-steps-for-eks-configuration)).

    - You can use either internal or public schema depending on your situation (with an internal scheme, [SCIM synchronization](../single-sign-on/scim/azure.md) with Azure AD will not work, only SSO Authorization).

    - ALB Configuration

        - HTTP/2 - `On`

        - Idle timeout - `4000 seconds`

## Recommended steps for EKS configuration

1. Deploy [EBS controller](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

    - Following the documentation, it is required to [create an IAM role](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html) and [deploy the EKS addon](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html).

    - It is recommended to use GP3 storages (to do this, you need to deploy a specific [StorageClass](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/862fe33d2174e00f449ac34859c9990340234a63/docs/parameters.md), [example](#configuration-examples)), but if required, you can use the default GP2 as well.

2. Deploy [ALB controller](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html).

    - Following the documentation, it is required to [create an IAM role](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html#lbc-helm-iam) and [deploy the ALB Controller](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html#lbc-helm-install).

    - If you need to use already created ALB, please review [this documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/use_cases/self_managed_lb/) and follow the steps below.

        The ALB routes each path of ODM straight to the Kubernetes Service that serves it, so each Service has a target group of its own. The Service names below are for the Helm release `odm`.

        | Service | HTTP path patterns | Health check path |
        | --- | --- | --- |
        | `odm-ui` | `/ui`, `/ui/*` | `/ui/` |
        | `odm-swagger` | `/swagger`, `/swagger/*` | `/swagger/helper/` |
        | `odm-user-docs` | `/user-docs`, `/user-docs/*` | `/user-docs/` |
        | `odm-mcp`, only with `mcp.enabled: true` | `/mcp`, `/mcp/*`, `/.well-known/oauth-authorization-server`, `/.well-known/oauth-protected-resource`, `/.well-known/oauth-protected-resource/*`, `/authorize`, `/token`, `/register`, `/consent`, `/auth/callback` | `/health` |
        | `odm-mailpit`, only with `mailpit.enabled: true` | `/mail`, `/mail/*` | `/mail/readyz` |
        | `odm-applications` | `/*`: every other path, the REST API and SCIM among them | `/actuator/health/readiness`, on port `8081` |

        1. [Create a target group manually](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-target-group.html) for each Service in the table, and specify **any** IP address from the EKS subnet, it will be automatically replaced with the actual address by the ALB controller.

            - Target type is `IP addresses`.

            - Target group name is any name. For example: `Genestack-ODM-ui`.

            - `Protocol : Port` is `HTTP : 80`

            - IP address type is `IPv4`

            - VPC is `YOUR_EKS_VPC`

            - Protocol version is `HTTP1`

            - Health check protocol is `HTTP`

            - Health check path is the Service's one in the table.

            - Health check port is the traffic port, except for `odm-applications`: override it with `8081`, the management port, where `odm-applications` serves its health.

        2. [Add rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-update-rules.html) to the HTTPS listener of the ALB. Every rule has the HTTP Host Header `ODM_FQDN`. `ODM_FQDN` - ODM url without `https://` or `http://`. For example: `odm.genestack.com`.

            - Forward the HTTP path patterns of each Service in the table to its target group. An ALB rule takes a limited number of path patterns ([quotas](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html)), so the paths of `odm-mcp` take several rules.

            - Give the rule of `odm-applications`, with `/*`, the last priority: it takes every path that the other rules do not. `/` and the pages of the UI before `/ui`, which old links still use, are among them: `odm-applications` answers them with a `301` redirect to their `/ui` pages, so the listener needs no redirect rules.

        3. Set the security headers in the attributes of the same listener:

            | Listener attribute | Value |
            | --- | --- |
            | `routing.http.response.strict_transport_security.header_value` | `max-age=31536000` |
            | `routing.http.response.x_content_type_options.header_value` | `nosniff` |
            | `routing.http.response.x_frame_options.header_value` | `SAMEORIGIN` |

        4. Set `ingress.enabled` parameter to `false` in ODM helm chart.

        5. Create and deploy a [TargetGroupBinding](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/use_cases/self_managed_lb/#create-targetgroupbinding-crd) for each target group ([example](#configuration-examples)) in Kubernetes.

        6. Ensure that health checks in each target group have passed (after ODM deployment). You may need to allow access for the ALB subnet in the EC2 Security Group, to port `8081` too for the health check of `odm-applications`.

    - If you don't have an ALB.

        - It will be created automatically during ODM deployment by ALB Controller.

        - Start the values from `odm/examples/ingress-alb.yaml` in the Helm chart. Its `ingress.annotations` set up the security headers above.

        - The controller creates a target group for each Service. Each target group takes its health check path from the `alb.ingress.kubernetes.io/healthcheck-path` annotation of its Service. The chart sets no annotations on the Services. The example sets this one, with the health check path in the table, in `<component>.service.annotations` of `applications`, `ui`, `swagger`, `userDocs`, `mailpit` and `mcp`. Without it the target group checks `/` on the traffic port. For `applications` the example also sets `alb.ingress.kubernetes.io/healthcheck-port: "8081"`, the management port.

3. [Optional] Deploy [External DNS](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md).

    - To automatically create records in Route53, it is recommended to use External DNS.

    - Be careful when using it, you should limit the list of zones with which it can work. And if the zone is used for something else, then make sure that it will not delete records from there.

## Cross-account S3 access

The approach is based on the [official documentation](https://repost.aws/knowledge-center/cross-account-access-s3) section `IAM policies and resource-based bucket policies`.

If the S3 bucket uses `SSE-KMS` encryption, then it is necessary to additionally grant access to the KMS key in the items below:

- in the `IAM policy` that is attached to the IAM role/user

- in the `Key policy` of the KMS key that is used to encrypt data in the S3 bucket

## Cross-account S3 access by assuming an IAM role in target account

The approach is based on the [official documentation](https://repost.aws/knowledge-center/cross-account-access-iam).

If the S3 bucket uses `SSE-KMS` encryption, then it is necessary to additionally grant access to the KMS key in the items below:

- in the `IAM policy` that is attached to the IAM role

- in the `Key policy` of the KMS key that is used to encrypt data in the S3 bucket

Choose the appropriate `Trust Policy` based on how the ODM accesses AWS resources:

- You can find the recommended `Trust Relationships` for EKS IRSA here [IRSA Trust Relationships](#configuration-examples).

- You can find the recommended `Trust Relationships` for EKS Pod Identity here [Pod Identity Trust Relationships](#configuration-examples).

This is how the `Trust Relationships` for the IAM role in the target account should appear - [IAM role in target account Trust Relationships](#configuration-examples)

## Cross-account ECR access

⚠️ **Mandatory in case of processors-controller usage**

The approach is based on the [official AWS documentation](https://repost.aws/knowledge-center/secondary-account-access-ecr) for ECR repository policies and cross-account access patterns.

ECR cross-account access requires configuring both:

- `IAM policy` attached to the IAM role/user

If the ECR repository uses `KMS encryption` with a customer-managed key, then it is necessary to additionally grant access to the KMS key in:

- The `IAM policy` that is attached to the IAM role/user

- The `Key policy` of the KMS key used to encrypt the ECR repository

## Configuration examples

<details><summary>AbortIncompleteMultipartUpload rule</summary>

```json
{
    "Rules": [
        {
            "ID": "PruneAbandonedMultipartUpload",
            "Status": "Enabled",
            "AbortIncompleteMultipartUpload": {
                "DaysAfterInitiation": 7
            }
        }
    ]
}
```

</details>

<details><summary>IAM policy for S3 access</summary>

```json
{
  "Version" : "2012-10-17",
  "Statement" : [
    {
      "Sid" : "ListObjectsInBucket",
      "Effect": "Allow",
      "Action" : [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Resource": ["S3_BUCKET_ARN"]
    },
    {
      "Sid" : "AllObjectActions",
      "Effect": "Allow",
      "Action" : [
        "s3:*Object*",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource" : ["S3_BUCKET_ARN/*"]
    },
    {
      "Sid" : "AllowUseOfTheKey",
      "Effect": "Allow",
      "Action" : [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource" : ["KMS_KEY_ARN"]
    }
  ]
}
```

</details>

<details><summary>IRSA Trust Relationships</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::AWS_ACCOUNT_ID:oidc-provider/oidc.eks.AWS_REGION.amazonaws.com/id/AWS_EKS_OIDC_ID"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "oidc.eks.AWS_REGION.amazonaws.com/id/AWS_EKS_OIDC_ID:sub": "system:serviceaccount:odm:odm",
                    "oidc.eks.AWS_REGION.amazonaws.com/id/AWS_EKS_OIDC_ID:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

</details>

<details><summary>Pod Identity Trust Relationships</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:TagSession",
                "sts:AssumeRole"
            ]
        }
    ]
}
```

</details>

<details><summary>IAM role in target account Trust Relationships</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "arn:aws:iam::AWS_ACCOUNT_ID:root"
            },
            "Action": [
                "sts:TagSession",
                "sts:AssumeRole"
            ]
        }
    ]
}
```

</details>

<details><summary>IAM policy for ECR access</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRRepositoryAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRImagePull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:DescribeRepositories",
        "ecr:DescribeImages",
        "ecr:ListImages"
      ],
      "Resource": "*"
    },
    {
      "Sid" : "AllowUseOfTheKey",
      "Effect": "Allow",
      "Action" : [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource" : ["KMS_KEY_ARN"]
    }
  ]
}
```

</details>

<details><summary>GP3 StorageClass example</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
parameters:
  type: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

</details>

<details><summary>TargetGroupBinding example</summary>

One TargetGroupBinding per target group. This one binds `odm-applications`; repeat it for each Service in the table, with the Service's name and the ARN of its target group.

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: odm-applications
  namespace: odm
spec:
  ipAddressType: ipv4
  serviceRef:
    name: odm-applications
    # Every Service the ALB routes to names its port http
    port: http
  targetGroupARN: TARGET_GROUP_ARN
```

</details>
