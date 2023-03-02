# Creating S3 bucket and authenticating with IRSA

[IRSA](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html) allows Pods running in Kubernetes to authenticate with IAM AWS using ServiceAccount identity. This is a great security advantage, as you don't create any permanent access keys, yet you have control over access security. General docs about how IRSA works in Giant Swarm clusters is available [here](https://docs.giantswarm.io/advanced/iam-roles-for-service-accounts/).

This document is based on [official AWS provider authentication doc](https://github.com/upbound/provider-aws/blob/main/AUTHENTICATION.md).

**Note**: This document is just a stub and not all the steps can be replicated
without cluster admin permissions. Please use [web identity](../s3-with-web-id/README.md) unless you really know you want IRSA approach.

## AWS setup

1. (Optional - you can use any already existing Police) Create a new IAM Policy that limits permissions to S3 and only to buckets meeting configured name pattern:

    ```shell
    aws iam create-policy \
      --policy-name $POLICY_NAME \
      --policy-document file://policy.json \
      --description "my custom policy for S3 access"
    ```

The [policy.json](policy.json) file contains IAM Policy that allows usage only of buckets starting with `harbor*` prefix.

1. Create a new IAM Role that references necessary IAM Policies and has the following `Trusted entities`:

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

    This Role definition allows IRSA to grant access to this Role to a `ServiceAccount` listed in `Condition`. This `ServiceAccount` has to be the `ServiceAccount` that your AWS crossplane provider `Pods` are using.

## Kubernetes setup

1. You have to edit the `ControllerConfig` object used by your AWS provider to include an annotation with a full ARN of the role you want to grant (created above). This should make crossplane to attach the same annotation to the `ServiceAccount` that is used to run the controller, as that is where we ultimately need it: `kubectl -n crossplane annotate controllerconfig upbound-provider-aws eks.amazonaws.com/role-arn=arn:aws:iam::[ACCOUNT_ID]:role/[ROLE_NAME]`. Check that this annotation is indeed carried over to the `ServiceAccount`.
1. Inspect the volumes section of the AWS provider pod and make sure that the `aws-iam-token` volume is injected: `k -n crossplane get po crossplane-59fb4694d6-zx8s8 -o jsonpath={.spec.volumes`. Restart the pod if it's not.
1. Create the IRSA `ProviderConfig`

    ```bash
    kubectl apply -f provider-upbound-aws-irsa.yaml
    ```

## Use it

You can now create a sample S3 bucket

```bash
kubectl apply -f s3_bucket.yaml
```
