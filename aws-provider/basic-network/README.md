# Basic VPC Network

This more advanced composition will create a standard 3 zone VPC utilising
6 subnets, 3 public and 3 private.

In order to achieve this, the composition uses two composition functions to
manage the resources.

- crossplane-contrib/function-go-templating
- crossplane-contrib/function-patch-and-transform

Both of these must be installed into your environment prior to creating a claim.

## Creating the claim

The claim is created against the VPC composition only using the `VpcNetworkClaim`.

The second composition `SubnetSet` is used by the VPC and normally would not
make sense to be applied isolated from the VPC.

### Claim parameters

- `deletionPolicy` One of `Delete`, `Orphan` details how crossplane treats
  resources when the claim is deleted
- `compositionSelector` Which composition to select when multiple compositions
  lie behind an XRD
- `region` The AWS region to build in
- `providerConfigRef` Which provider config to use to build the infrastructure
- `vpcCidr` This is the CIDR for the VPC. It must be large enough to encompass
  6 subnets with a minimum of `/28` (16 IP addreses) - See [VPC CIDR Blocks]
- `subnets` Contains information on where to build the subnets (which zones) and
  at what size
  - `zones` A list of 3 availability zones to build in
  - `cidrBlocks` A list of 6 cidr blocks. The first 3 will be public and the
    last 3 will be private
- `tags` Additional tags to apply to all resources. Do not add `Name` here, this
  will be automatically determined from the resource type, and location where
  appropriate.

The name of the VPC will be taken from the name of the Claim coupled with the
region it is found in.

Labels applied to the claim will also be merged in to the tags. This excludes
any `kubernetes.io` tags - this seems to be a limitation of crossplane rather
than a limitation of the composition.

### Installation

Apply all resources under the [xrd](./xrd) folder, then apply the claim.

```nohighlight
$ crossplane beta trace vpcnetworkclaim myapp-basic-vpc

NAME                                                                   SYNCED   READY   STATUS
VpcNetworkClaim/myapp-basic-vpc (default)                              -        -
└─ VpcNetwork/myapp-basic-vpc-xxtgj                                    True     True    Available
   ├─ DefaultSecurityGroup/myapp-basic-vpc-eu-central-1                True     True    Available
   ├─ EIP/myapp-basic-vpc-ngw-eu-central-1a                            True     True    Available
   ├─ EIP/myapp-basic-vpc-ngw-eu-central-1b                            True     True    Available
   ├─ EIP/myapp-basic-vpc-ngw-eu-central-1c                            True     True    Available
   ├─ InternetGateway/myapp-basic-vpc-eu-central-1                     True     True    Available
   ├─ NATGateway/myapp-basic-vpc-eu-central-1a                         True     True    Available
   ├─ NATGateway/myapp-basic-vpc-eu-central-1b                         True     True    Available
   ├─ NATGateway/myapp-basic-vpc-eu-central-1c                         True     True    Available
   ├─ Route/myapp-basic-vpc-igw-eu-central-1a                          True     True    Available
   ├─ Route/myapp-basic-vpc-igw-eu-central-1b                          True     True    Available
   ├─ Route/myapp-basic-vpc-igw-eu-central-1c                          True     True    Available
   ├─ Route/myapp-basic-vpc-ngw-eu-central-1a                          True     True    Available
   ├─ Route/myapp-basic-vpc-ngw-eu-central-1b                          True     True    Available
   ├─ Route/myapp-basic-vpc-ngw-eu-central-1c                          True     True    Available
   ├─ VPC/myapp-basic-vpc-eu-central-1                                 True     True    Available
   ├─ SubnetSet/myapp-basic-vpc-private                                True     True    Available
   │  ├─ RouteTableAssociation/myapp-basic-vpc-private-eu-central-1a   True     True    Available
   │  ├─ RouteTableAssociation/myapp-basic-vpc-private-eu-central-1b   True     True    Available
   │  ├─ RouteTableAssociation/myapp-basic-vpc-private-eu-central-1c   True     True    Available
   │  ├─ RouteTable/myapp-basic-vpc-private-eu-central-1a              True     True    Available
   │  ├─ RouteTable/myapp-basic-vpc-private-eu-central-1b              True     True    Available
   │  ├─ RouteTable/myapp-basic-vpc-private-eu-central-1c              True     True    Available
   │  ├─ Subnet/myapp-basic-vpc-private-eu-central-1a                  True     True    Available
   │  ├─ Subnet/myapp-basic-vpc-private-eu-central-1b                  True     True    Available
   │  └─ Subnet/myapp-basic-vpc-private-eu-central-1c                  True     True    Available
   └─ SubnetSet/myapp-basic-vpc-public                                 True     True    Available
      ├─ RouteTableAssociation/myapp-basic-vpc-public-eu-central-1a    True     True    Available
      ├─ RouteTableAssociation/myapp-basic-vpc-public-eu-central-1b    True     True    Available
      ├─ RouteTableAssociation/myapp-basic-vpc-public-eu-central-1c    True     True    Available
      ├─ RouteTable/myapp-basic-vpc-public-eu-central-1a               True     True    Available
      ├─ RouteTable/myapp-basic-vpc-public-eu-central-1b               True     True    Available
      ├─ RouteTable/myapp-basic-vpc-public-eu-central-1c               True     True    Available
      ├─ Subnet/myapp-basic-vpc-public-eu-central-1a                   True     True    Available
      ├─ Subnet/myapp-basic-vpc-public-eu-central-1b                   True     True    Available
      └─ Subnet/myapp-basic-vpc-public-eu-central-1c                   True     True    Available
```

[VPC CIDR Blocks]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html
