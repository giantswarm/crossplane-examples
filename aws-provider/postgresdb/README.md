# `crossplane` PostgresDB example

In this more complex example, we're going to walk through the process of
enabling provider-families with `sts:AssumeRoleWithWebIdentity` to the
management cluster, then using that to bootstrap a database using a secondary
role assumed by crossplane for the actual build.

> **Warning** Creating IAM permissions
>
> The  following document assumes that you have permissions to create IAM roles
> and policies inside your account.
>
> If you do not have such permissions, please reach out to your account
> administrator and ask them to create these for you.

## Create the primary role

1. Edit the file [`crossplane-assume-role-policy`](./policies/crossplane-assume-role-policy.json)
   and set the variable `${AWS_ACCOUNT_ID}` to the ID of the AWS account you're
   going to use.

1. Apply this policy to AWS

   ```bash
   NAME="crossplane-assume-role-policy"
   aws iam create-policy \
     --policy-name $POLICY_NAME \
     --policy-document file://${POLICY_NAME}.json \
     --description "Custom policy for assuming roles"
   ```

1. Edit the trust policy for this role to include any additional service accounts

   The following command modifies the policy [`crossplane-assume-role-trust-policy.json`](./policies/crossplane-assume-role-trust-policy.json).
   If you already know the name of the service account you wish to use, you
   may feel more comfortable editing this file manually.

   > **Note** Service account listings
   >
   > In the trust policy for this role, I am specifically mentioning all
   > service accounts for the providers I require access to the account.
   >
   > Whilst this is a concious decision to provide visibility within the trust
   > about which service accounts are allowed to connect, you may wish to
   > allow any service account in a given namespace to connect as long as it
   > starts with a provider prefix.
   >
   > In this instance, replace the entire `Condition` statement with an
   > equivelant [`StringLike`](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_String)
   > condition.

   The following example uses the `sponge` command from the `moreutils` package
   to compensate for `jq` not having an `--inplace` flag. This will likely not
   exist on your system but can normally be installed with the package manager
   of your choice.

   ```bash
   PROVIDER_NAME=my-other-provider
   jq '.Statement[].Condition."ForAnyValue:StringEquals"."${OIDC_PROVIDER_URL}:sub" |=
       ( .+ ["system:serviceaccount:crossplane:'$(
           kubectl -n crossplane get ControlLerConfig ${PROVIDERNAME} -o \
               jsonpath='{.spec.serviceAccountName}'
       )'"] | unique)' policies/crossplane-assume-role-trust-policy.json \
           | sed '/".*:",/d' | sponge policies/crossplane-assume-role-trust-policy.json
   ```

1. Next, create a new WebIdentity role called `crossplane-assume-role` and bind
   this policy to it. The Open ID Connect Identity Provider should be set to the
   one for your cluster.

   - substitute the variables `AWS_ACCOUNT_ID` and `OICD_PROVIDER_URL` in the
     trust policy:

     ```bash
      export OIDC_PROVIDER_URL=$(aws iam list-open-id-connect-providers \
          | jq '.OpenIDConnectProviderList[] | .Arn | split("/")[1] | select(. | test("^\\w+.mycluster.(?!mycluster)"))')
      export AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -rn .Account)
      cat policies/crossplane-assume-role-trust-policy.json | envsubst \
          | sponge policies/crossplane-assume-role-trust-policy.json
     ```

   - Check that the role does not already exist

     ```bash
     aws iam list-roles | jq '.Roles[] | select( .RoleName == "crossplane-assume-role")'
     ```

     If the role does not exist, we need to create it and attach the policy

     ```bash
       aws iam create-role --role-name crossplane-assume-role --policy-document file://policies/crossplane-assume-role-trust-policy.json
       aws iam attach-role-policy --policy-arn \
           "arn:aws:iam::aws:policy/crossplane-assume-role-policy" \
           --role-name crossplane-assume-role
     ```

     If the role already exists, update its trust policy with

     ```bash
       aws iam update-assume-role-policy --role-name crossplane-assume-role \
           --policy-document file://policies/crossplane-assume-role-trust-policy.json
     ```

## Create the secondary role

Use the steps defined in [creating the primary role](#create-the-primary-role) to
create a new role `rds-crossplane-role`

This second role will use the attach the trust policy and IAM permissions policy

- [rds-crossplane-policy](./policies/rds-crossplane-policy.json).
- [rds-crossplane-role-trust-policy](./policies/rds-crossplane-role-trust-policy.json)

> **Note** Creating IAM policies to be assumed by other roles
>
> When creating roles to be assumed by the service account identity role
> `crossplane-assume-role`, at the very least, the policy must include the
> following statement for other roles to be allowed to assume it.
>
> ```json
> {
>     "Sid": "VisualEditor2",
>     "Effect": "Allow",
>     "Action": "sts:AssumeRole",
>     "Resource": "*",
>     "Condition": {
>         "ForAnyValue:StringLike": {
>             "aws:PrincipalArn": [
>                 "arn:aws:sts::${AWS_ACCOUNT_ID}:role/crossplane-assume-role",
>                 "arn:aws:sts::${AWS_ACCOUNT_ID}:assumed-role/rds-crossplane-role*"
>             ]
>         }
>     }
> }
> ```
>
> Without this, crossplane may not be able to assume the role and you will
> experience permissions failures when building resources.

The policy file [rds-crossplane-policy](./policies/rds-crossplane-policy.json)
is a restrictive policy that allows only the specific actions required to build
this example.

## Enabling providers

Next, we need to set up the providers.

In your management clusters git repository, go to the `management-clusters/MC_NAME/crossplane-providers`
folder and create `kustomization.yaml` file in a sub-directory named after the
apigroup of your provider. For example:

```nohighlight
├── crossplane-providers
│   ├── ec2
│   │   └── kustomization.yaml
│   ├── kms
│   │   └── kustomization.yaml
│   └── rds
│       └── kustomization.yaml
```

into each `kustomization.yaml` file, copy the following contents:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/giantswarm/management-cluster-bases//extras/crossplane/providers/upbound/aws?ref=main
patches:
  - patch: |
      - op: replace
        path: /metadata/name
        value: provider-aws-${PROVIDER}
      - op: replace
        path: /spec/serviceAccountName
        value: upbound-provider-aws-${PROVIDER}
    target:
      kind: ControllerConfig
  - patch: |
      - op: replace
        path: /metadata/name
        value: provider-aws-${PROVIDER}
      - op: replace
        path: /spec/controllerConfigRef/name
        value: provider-aws-PROIVIDER
      - op: replace
        path: /spec/package
        value: xpkg.upbound.io/upbound/provider-aws-${PROVIDER}:${VERSION}
    target:
      kind: Provider
  - patch: |
      - op: add
        path: /metadata/name
        value: upbound-provider-aws-${PROVIDER}
      - op: add
        path: /metadata/annotations/eks.amazonaws.com~1role-arn
        value: arn:aws:iam::${AWS_ACCOUNT_ID}:role/crossplane-assume-role
    target:
      kind: ServiceAccount
  - patch: |
     - op: replace
       path: /metadata/name
       value: crossplane-use-psp-upbound-provider-aws-${PROVIDER}
     - op: replace
       path: /subjects/0/name
       value: upbound-provider-aws-${PROVIDER}
    target:
      kind: ClusterRoleBinding
```

The important section in this patch is the role annotation given to the service
account.

```yaml
      - op: add
        path: /metadata/annotations/eks.amazonaws.com~1role-arn
        value: arn:aws:iam::${AWS_ACCOUNT_ID}:role/crossplane-assume-role
```

This patch allows the following to be injected into your pods environment and
the service account token file to be injected.

```bash
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
AWS_STS_REGIONAL_ENDPOINTS=regional
AWS_ROLE_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:role/crossplane-assume-role
```

From the `crossplane-providers` directory, run the following script:

```bash
export VERSION=v0.37.0
for directory in $(find . -type d -not -name '.' -printf '%f\n'); do
    export PROVIDER="${directory}"
    cat "${directory}/kustomization.yaml" | envsubst | sponge "${directory}/kustomization.yaml"
done
```

> **Note** Applying manifests
>
> If you are using this on a cluster away from Giant Swarm, you may wish to
> render these manifests individually.
>
> For this, you will need `kustomize` installed.
>
> Inside each directory, build the `Kustomization` and optionally, feed the
> output to `yq` and have the resulting manifest split into multiple files.
>
> ```bash
> $ kustomize build . | yq -s .kind
> $ tree
> .
> ├── ClusterRoleBinding.yml
> ├── ControllerConfig.yml
> ├── kustomization.yaml
> ├── Provider.yml
> └── ServiceAccount.yml
> ```
>
> You're free to delete the `kustomization.yaml` file at this point and apply
> the resulting changes in a manner of your choice.
>
> It should, however be noted that the cluster role binding refers to `ClusterRole`
> you may not have on your cluster. That `ClusterRole` is available as part of
> our [crossplane helm chart](https://github.com/giantswarm/crossplane/tree/main/helm/crossplane).

## Installing the database

Once our providers are installed, we are able to apply the files necessary for
building our database.

1. Create a secret for the database

   ```bash
   kubectl create secret generic -n default aws-rds-db-pass \
     --from-literal=database-password=$(pwgen -sn 20 1)
   ```

1. Apply the contents of the `xrd` folder.

   ```bash
   kubectl apply -f xrd
   ```

1. Prepare and apply provider config

   The file [`providerconfig.yaml`](./providerconfig.yaml) contains the
   instruction between the provider your composition. This needs to have the
   account ID injected into it and then be applied to the cluster.

   ```bash
   export AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -rn .Account)
   cat providerconfig.yaml | envsubst | kubectl apply -f -
   ```

   In this file, we need to define two sets of credentials. The first is the
   WebIdentityAssumeRole.

   Although we've already applied this to the pod, we need to instruct crossplane
   that it's to use it as its primary login and we do this under the `credentials`
   property.

   The second role is the role we'll use to build the database.

1. Edit [`claim.yaml`](./claim.yaml)

   The file `claim.yaml` contains the defined values for our database and the
   cluster VPC we need to connect back to. Many of the values in this file are
   already set with (reasonably) sensible defaults whilst others are masked out
   with "PATCHED BY KUSTOMIZE"

   Fill out any values you need to change, refer to the [`definition.yaml`](./xrd/definition.yaml)
   if you're unsure of the meaning of any value.

    Once you're happy with the results, we can apply this to the cluster with

    ```bash
    kubectl apply -f claim.yaml
    ```

It will take about 5 - 10 minutes for the database to build. Go get yourself a
coffee, then when you check back, we should be able to connect to the database.

## Testing

1. Apply the pod file

   ```bash
   kubectl apply -f ../../azure-provider/postgresdb/pgclient.yaml
   ```

2. Connect to the database

   ```bash
   DB_USR=$(yq -M .spec.parameters.db.admin claim.yaml)
   DB_PASS=$(kubectl -n default get secret backstage-db-pass -o yaml | yq '.data.database-password | @base64d')
   DB_SVR=$(kubectl -n default get instances.rds.aws.upbound.io -o yaml | yq -M .items[0].status.atProvider.address)
   DB=$(yq -M .spec.parameters.db.name claim.yaml)
   kubectl -n default exec -it postgresql-client -- psql postgresql://${DB_USR}:${DB_PASS}@${DB_SVR}.:5432/${DB}?sslmode=require

   psql (14.5, server 14.7)
   SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
   Type "help" for help.

   myapp=>
   ```
