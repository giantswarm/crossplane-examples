
# AWS RDS management

This example shows how to create a dedicated [Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) and a [User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) that has permissions restricted to only RDS instances having a configured prefix. Then, it sets up Crossplane's `ProviderConfig` and creates a RDS instance using Crossplane's AWS Provider.

The example uses [IAM Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) for authentication. This forces the user to manage and ensure security of Access Keys. If your cluster runs in AWS, please check our [other examples](../../README.md) that use Web Identity based approach, as Access Keys are not recommended in this case.

## AWS setup

Make sure you're logged in to your AWS account with the `aws` tool.

We're assuming that you have the following set:

```bash
POLICY_NAME=test-rds-access
USER_NAME=test-rds-access
```

1. Create a new IAM Policy that limits permissions to RDS and only to RDS instances meeting configured name pattern:

    ```shell
    aws iam create-policy \
      --policy-name $POLICY_NAME \
      --policy-document file://policy.json \
      --description "my custom policy for RDS access"
    ```

The [policy.json](policy.json) file contains IAM Policy that allows usage only of RDS instances starting with `test*` prefix.

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
      name: aws-rds-access
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

As the simplest use case possible, we request creation of a new RDS instance using a manifest like:

```yaml
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata:
  annotations:
    upjet.upbound.io/manual-intervention: This resource has a password secret reference.
  name: test-mysqldb
  namespace: org-test
spec:
  forProvider:
    allocatedStorage: 20
    autoMinorVersionUpgrade: true
    backupRetentionPeriod: 14
    backupWindow: 09:46-10:16
    engine: mysql
    engineVersion: "8.0"
    instanceClass: db.t2.micro
    maintenanceWindow: Mon:00:00-Mon:03:00
    name: test-mysql
    passwordSecretRef:
      key: password
      name: test-dbinstance
      namespace: org-test
    publiclyAccessible: false
    region: eu-central-1
    skipFinalSnapshot: true
    storageEncrypted: false
    storageType: gp2
    username: adminuser
  writeConnectionSecretToRef:
    name: mysql-db-out-metadata
    namespace: org-test
---
apiVersion: v1
kind: Secret
metadata:
  name: test-dbinstance
  namespace: org-test
stringData:
  password: test01-Password
```

You can apply it using a ready file from repo:

```bash
kubectl apply -f rds.yaml
```
