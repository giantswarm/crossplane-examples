# crossplane-examples

This repository contains examples showing how to use crossplane.

## Tools

Our examples use the following examples - please make sure you have them installed:

- [`jq`](https://stedolan.github.io/jq/)
- [`yq`](https://mikefarah.gitbook.io/yq/)
- `envsubst` from package `gettext`

## Index of examples

- [Azure Provider](azure-provider/README.md)
  - simple [Storage Blob Containers](azure-provider/storage-blob/README.md)
  - more advanced [Postgres DB as a part of bigger composition](azure-provider/postgresdb/README.md)
- [AWS Provider](aws-provider/README.md)
  - different authentication modes showed using simple S3 bucket
    - web identity based (recommended approach): [S3 bucket using web identity](aws-provider/s3-with-web-id/README.md)
    - legacy approach (not recommended if running in AWS): [S3 bucket using access keys](aws-provider/s3-with-access-keys/README.md)
    - IRSA (not recommended, as it limits possible Roles to just 1): [S3 bucket using IRSA authX](aws-provider/s3-with-irsa/README.md)
