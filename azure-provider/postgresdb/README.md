# crossplane-azure-pgsql-example

This is an example of how you might add postgres to azure using crossplane

This example will build the following resources

- `ResourceGroup`
- `VirtualNetwork`
- `Subnet` with label public
  unused in this example but may contain public facing, non-kubernetes workloads.
- `Subnet` with label private delegated to Postgres.
  Nothing else may go into this subnet
- `PrivateDNSZone` for Postgres server to register with
- `PrivateDNSZoneVirtualNetworkLink` connecting the private dns zone to
  the virtual network
- `FlexibleServer` Server running postgres database
- `Database`
- `FlexibleServerFirewallRule` allows access to the server from other networks
- `VirtualNetworkPeering` Peering connections between the created VirtualNetwork
  and the Kubernetes cluster VirtualNetwork

## Files and what they do

- File: [`xrd/xrc.yaml`](./xrd/xrc.yaml) `CompositeResourceComposition` - Defines
  what resources will be built by this template.
- File: [`xrd/xrd.yaml`](./xrd/xrd.yaml) `CompositeResourceDefinition` - Defines
  template properties that can be set by the user.
- File: [`claim.yaml`](./claim.yaml) Creates a claim against the template using
  values required for this instance.
- File: [`providerconfig.yaml`](./providerconfig.yaml) Configures the provider
  used by the claim.
- File: [`tcap-sql.json`](./tcap-sql.json) A sample IAM role restricting the
  crossplane provider service principle to only the resources required for the
  build.

## Installation

Make sure you are logged in to your Azure account with the `az` tool and you know
your subscription ID as well as the `ResourceGroup` and `VirtualNetwork` names
of the cluster you will be connecting to. In this example they will be called
`MyResourceGroup` and `MyVnet`

## Azure setup

Before we can make a claim against the definition, certain steps need to be
undertaken to set up crossplane for this build.

1. Create the role and set up the service principle.

   ```bash
   export AZURE_SUBSCRIPTION_ID=$(az account show | jq -r .id)

   az role definition create --role-definition "$(cat tcap-sql.json | envsubst)"

   az ad sp create-for-rbac -n crossplane-test --sdk-auth --role tcap-sql \
       --scopes="/subscriptions/${AZURE_SUBSCRIPTION_ID}" > crossplane-azure-provider-key.json
   ```

## Kubernetes setup

1. Create secret for the database

   This step will create a password consisting of uppercase, lowercase and numbers
   for your database and store this as a kubernetes secret. Special characters are
   ignored for this example due to limitations caused by URL strings.

   ```bash
   k create secret generic -n default azure-db-pass \
     --from-literal=database-password=$(pwgen -sn 20 1)
   ```

2. Create secret containing service principle account details

   We need to store the service account principle credentials for crossplane to use.

   To do this, we create a secret containing the entire JSON credentials file.

   ```bash
   k create secret generic -n default azure-account-creds \
     --from-file=credentials=crossplane-azure-provider-key.json
   ```

## Crossplane setup

1. Create provider config

   ```bash
   k apply -f providerconfig.yaml
   ```

2. apply both `xrd/xrc.yaml` and `xrd/xrd.yaml`

   ```bash
   k apply -f xrd
   ```

3. Edit the claim and fill out all values you require then apply it.

   In this example, some values in the claim have been templated for substitution
   via environment variables using the `envsubst` command. This helps keep *secure*
   values out of source control

   ```bash
   export AZURE_RG=MyResourceGroup
   export AZURE_VNET=MyVnet
   export AZURE_SUBSCRIPTION_ID=$(az account show | jq -r .id)
   cat claim.yaml | envsubst | k apply -f -
   ```

## Testing

1. Apply test pod template

   ```bash
   k apply -f pgclient.yaml
   ```

2. Wait for resources to come up.

   This may take anything from 3 to 5 minutes

3. Connect to the database from the test pod

   ```bash
   export DB_SVR=$(yq .spec.name claim.yaml)-svr
   export DB=$(yq .spec.name claim.yaml)
   export DB_USR=$(yq .spec.parameters.db.admin claim.yaml)

   k exec -it postgresql-client -- psql postgresql://${DB_USR}:$(kubectl get \
       secret azure-db-pass -o yaml | \
       yq '.data.database-password | @base64d')@${DB_SVR}.postgres.database.azure.com:5432/${DB}?sslmode=require
   ```
