# Azure storage container and blob access

This example shows how to create a separate [storage container](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction#containers) with a dedicated [Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals#service-principal-object) that has permissions restricted to this storage container. Then it sets up Crossplane's `ProviderConfig` and creates a storage container using Crossplane's Azure Provider.

## Azure setup

Make sure you're logged in to your Azure account with the `az` tool, and you know your subscription ID and an existing Resource Group that you want the storage account to be created in. We're assuming the Resource Group is named `MyResourceGroup`.

1. Create a new storage account within our resource group ([reference](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create)):

    ```bash
    az storage account create -n mystorageaccount -g MyResourceGroup -l westus --sku Standard_LRS
    ```

1. Now, let's create a Role that is limited to accessing the new storage account. The role is available in <role.json> file:

    ```json
    {
        "Name": "mystorageaccount-access",
        "Description": "custom role for accessing containers in mystorageaccount",
        "AssignableScopes": [
            "/subscriptions/[SUBSCRIPTION ID]/resourcegroups/MyResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount"
        ],
        "Actions": [
            "Microsoft.Storage/storageAccounts/blobServices/containers/delete",
            "Microsoft.Storage/storageAccounts/blobServices/containers/read",
            "Microsoft.Storage/storageAccounts/blobServices/containers/write"
        ],
        "DataActions": [
            "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/delete",
            "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read",
            "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write",
            "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/move/action",
            "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/add/action"
        ]
    }
    ```

    ```bash
    az role definition create --role-definition "$(cat role.json)"
    ```

1. Add Service Principle that will be granted this Role and will be used by Crossplane. We save SP access keys to a temporary file.

    ```bash
    az ad sp create-for-rbac -n crossplane-blob-access  --sdk-auth --role mystorageaccount-access --scopes="/subscriptions/[SUBSCRIPTION ID]/resourcegroups/MyResourceGroup/providers/Microsoft.Storage/storageAccounts/mystorageaccount" > /tmp/keys.json
    ```

1. Grant additional permissions to the SP, so it can list Storage Account within our Resource Group:

    ```bash
    READER_ROLE_ID=$(az role definition list -n "Reader and Data Access" | jq -r '.[0].name')
    SP_ID=$(az ad sp list --display-name "gremlin-harbor-blob-access" | jq -r '.[0].id')
    
    az role assignment create --assignee-object-id $SP_ID --assignee-principal-type ServicePrincipal --role $READER_ROLE_ID --scope /subscriptions/[SUBSCRIPTION ID]/resourcegroups/MyResourceGroup
    ```

## Kubernetes setup

1. We need to save access keys for Azure Service Principle as a `Secret`:

    ```bash
    kubectl create secret generic azure-blob-access  --from-file=credentials=/tmp/keys.json
    ```

1. Now we can create a `ProviderConfig` that uses the `Secret` created above:

    ```bash
    kubectl apply -f providerconfig.yaml
    ```

## Use it

As the simples use case possible, we request creation of a new storage container using a manifest like:

    ```yaml
    apiVersion: storage.azure.upbound.io/v1beta1
    kind: Container
    metadata:
      name: example
    spec:
      forProvider:
        containerAccessType: private
        storageAccountName: mystorageaccount
      providerConfigRef:
        name: test-harbor-blob-storage
    ```

    You can apply it using a ready file from repo:

    ```bash
    kubectl apply -f container-resource.yaml
    ```
