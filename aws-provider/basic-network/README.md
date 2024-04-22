# Basic VPC Network

This more advanced composition will create a standard 3 zone VPC utilising
6 subnets, 3 public and 3 private.

In order to achieve this, the composition uses two composition functions to
manage the resources.

- crossplane-contrib/function-go-templating
- crossplane-contrib/function-patch-and-transform

Both of these must be installed into your environment prior to creating a claim
against the composition.

Claims may be made by providing the following varibles

- `deletionPolicy` How to handle resources when deleting the claim.
  One of `Orphan` or `Delete`
- `name` This is the name of the VPC to build. All resources will be prefixed
  with this value for easy identification inside the cloud console
- `labels` A set of labels to apply to kubernetes manifests
- `compositionSelector` - For selecting a different composition to use with the
  definition when multiple compositions match.
- `region`
