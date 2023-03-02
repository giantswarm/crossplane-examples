# Creating S3 bucket and authenticating with Web Identity

This approach is very similar to the [IRSA one](../s3-with-irsa/README.md) and also uses OIDC provider. Still, it has two important advantages over direct IRSA one:

- allows to setup both Kubernetes and AWS side of it without full cluster admin permissions
- allows you to use multiple `ProviderConfigs` referring different IAM Roles and then use Kubernetes RBAC to grant permissions to use them to your users.

This document is based on [official AWS provider authentication doc](https://github.com/upbound/provider-aws/blob/main/AUTHENTICATION.md).

## AWS setup

1. (Optional - you can use any already existing Police) Create a new IAM Policy that limits permissions to S3 and only to buckets meeting configured name pattern:

    ```shell
    aws iam create-policy \
      --policy-name $POLICY_NAME \
      --policy-document file://policy.json \
      --description "my custom policy for S3 access"
    ```

The [policy.json](policy.json) file contains IAM Policy that allows usage only of buckets starting with `harbor*` prefix.

1. Create a new IAM Role that allows usage of necessary IAM Policies and has the following `Trusted entities` (this has to point to your OIDC Identity Provider):

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::[ACCOUNT_ID]:oidc-provider/[OIDC_PROVIDER_DOMAIN_OF_YOUR_CLUSTER]"
                },
                "Action": "sts:AssumeRoleWithWebIdentity",
                "Condition": {
                    "StringEquals": {
                        "[OIDC_PROVIDER_DOMAIN_OF_YOUR_CLUSTER]:sub": "system:serviceaccount:[K8S_NAMESPACE]:[K8S_SERVICE_ACCOUNT]"
                    }
                }
            }
        ]
    }
    ```

    This Role definition allows crossplane's AWS provider to use this Role if a request to use it comes from a `ServiceAccount` listed in `Condition`. This `ServiceAccount` has to be the `ServiceAccount` that your AWS crossplane provider `Pods` are using. For example, if you're using the official Upbound AWS provider, you can get the `ServiceAccount` like this:

    ```bash
    kubectl -n crossplane get po -l pkg.crossplane.io/provider=provider-aws -o jsonpath='{.items[0].spec.serviceAccount}'
    ```

## Kubernetes setup

1. Create the IRSA `ProviderConfig`

    ```bash
    kubectl apply -f provider-upbound-aws-webid.yaml
    ```

    It has reference the correct IAM Role using its full ARN, so you need to fill in the following template:

    ```yaml
    apiVersion: aws.upbound.io/v1beta1
    kind: ProviderConfig
    metadata:
      name: webid
    spec:
      credentials:
        source: WebIdentity
        webIdentity:
          roleARN: arn:aws:iam::[ACCOUNT_ID]:role/[ROLE_NAME]
    ```

## Use it

You can now create a sample S3 bucket using the `ProviderConfig` with limited permissions:

```bash
kubectl apply -f s3_bucket.yaml
```
