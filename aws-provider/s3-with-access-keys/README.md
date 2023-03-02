
# AWS S3 storage access

This example shows how to create a dedicated [Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) and a [User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) that has permissions restricted to only S3 buckets having a configured prefix. Then, it sets up Crossplane's `ProviderConfig` and creates a S3 bucket using Crossplane's AWS Provider.

The example uses [IAM Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) for authentication. This forces the user to manage and ensure security of Access Keys. If your cluster runs in AWS, please check our [other examples](../../README.md) that use Web Identity based approach, as Access Keys are not recommended in this case.

## AWS setup

Make sure you're logged in to your AWS account with the `aws` tool.

We're assuming that you have the following set:

```bash
POLICY_NAME=harbor-s3-access
USER_NAME=harbor-s3-access
```

1. Create a new IAM Policy that limits permissions to S3 and only to buckets meeting configured name pattern:

    ```shell
    aws iam create-policy \
      --policy-name $POLICY_NAME \
      --policy-document file://policy.json \
      --description "my custom policy for S3 access"
    ```

The [policy.json](policy.json) file contains IAM Policy that allows usage only of buckets starting with `harbor*` prefix.

1. Create a new User and attach the above policy to it:

    ```shell
    create-user --user-name $USER_NAME
    POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`$POLICY_NAME`].Arn' --output text)
    attach-user-policy --user-name $USER_NAME --policy-arn $POLICY_ARN
    ```

1. Create User access keys. **The secret key will be shown only once**. Please make sure you securely note/save the new access keys:

```bash
aws iam create-access-key --user-name $USER_NAME
```

## Kubernetes setup

1. We need to save access keys for our new User as a `Secret`:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Secret
    metadata:
      name: aws-s3-access
      namespace: default
    stringData:
      creds: |
        $(printf "[default]\n    aws_access_key_id = %s\n    aws_secret_access_key = %s" "${AWS_ACCESS_KEY_CREATED_ABOVE}" "${AWS_SECRET_KEY_CREATED_ABOVE}")
    EOF
    ```

1. Now we can create a `ProviderConfig` that uses the `Secret`:

    ```bash
    kubectl apply -f providerconfig.yaml
    ```

## Use it

As the simplest use case possible, we request creation of a new S3 storage bucket using a manifest like:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: harbor-test-bucket
spec:
  deletionPolicy: Delete
  forProvider:
    region: eu-west-2
  providerConfigRef:
    name: aws-s3-access
```

You can apply it using a ready file from repo:

```bash
kubectl apply -f s3_bucket.yaml
```
