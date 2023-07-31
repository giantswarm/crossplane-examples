# IAM permissions for creating an RDS database

The file [`rds-crossplane-policy`] contains the permissions required to build
the [aws/postgresdb](../README.md) example.

To accomodate building a full networking stack and peering with the cluster
VPC, a generally wider set of permissions are required to encompass build.

This document breaks down what those permissions are, and why they are
necessary.

Broadly speaking, each group is split into 3 sections.

- **Read** which may encompass both `Describe` and `List`
- **Write** which encompasses `Create`, `Modify`, `Replace`, `Associate`,
  `Add`, `Accept`
- **Delete** which also encompasses `Revoke`, `Remove`, `Dissassociate`

## EC2

For all permissions in the `ec2` category, the following permissions are also
required:

- `ec2:Describe*` Enable all read operations on EC2 resources including vpcs,
  subnets, security groups, peering connections, routetables, routes.
- `ec2:CreateTags` Allow crossplane to create tags on resources in the `ec2`
  group.
- `ec2:DeleteTags` Allow crossplane to delete tags in on resources in the `ec2`
  group.

## VPC

To create and manage a VPC, the following permissions are required.

- `ec2:CreateVpc`
- `ec2:DeleteVpc`
- `ec2:DisassociateVpcCidrBlock`
- `ec2:ModifyVpcAttribute`

## Subnets

To create and manage subnets, the following permissions are required

- `ec2:CreateSubnet`
- `ec2:DeleteSubnet`
- `ec2:DisassociateSubnetCidrBlock`

## Peering

Peering connections require acceptance as well as create/modify/delete
permissions. We may also require the capability of rejecting a peering
connection in certain instances.

- `ec2:AcceptVpcPeeringConnection`
- `ec2:CreateVpcPeeringConnection`
- `ec2:DeleteVpcPeeringConnection`
- `ec2:ModifyVpcPeeringConnectionOptions`
- `ec2:RejectVpcPeeringConnection`

## Routing

In order to allow the VPC peering connection to be accessed from the
individual subnets, we need to create routes and route tables inside the RDS
VPC, as well as routes on existing routetables in the cluster VPC.

- `ec2:AssociateRouteTable`
- `ec2:CreateRoute`
- `ec2:CreateRouteTable`
- `ec2:DeleteRoute`
- `ec2:DeleteRouteTable`
- `ec2:DisassociateRouteTable`
- `ec2:ReplaceRoute`
- `ec2:AssociateRouteTable`
- `ec2:CreateRoute`
- `ec2:CreateRouteTable`
- `ec2:DeleteRoute`
- `ec2:DeleteRouteTable`
- `ec2:DisassociateRouteTable`
- `ec2:ReplaceRoute`

## Key management

To encrypt the database we need a managed key inside KMS. These permissions
grant us the rights to create and manage this through crossplane.

- `kms:CreateAlias`
- `kms:CreateGrant`
- `kms:CreateKey`
- `kms:Decrypt`
- `kms:DeleteAlias`
- `kms:DescribeKey`
- `kms:DisableKey`
- `kms:EnableKey`
- `kms:Encrypt`
- `kms:GetKeyPolicy`
- `kms:GetKeyRotationStatus`
- `kms:ListAliases`
- `kms:ListKeys`
- `kms:ListResourceTags`
- `kms:PutKeyPolicy`
- `kms:RevokeGrant`
- `kms:ScheduleKeyDeletion`
- `kms:TagResource`
- `kms:UntagResource`

## Secrets management

These permissions are required by the RDS database to manage the password for
the database once inside AWS including creation, tagging rotation and deletion.

- `secretsmanager:CreateSecret`
- `secretsmanager:DeleteSecret`
- `secretsmanager:PutSecretValue`
- `secretsmanager:RotateSecret`
- `secretsmanager:TagResource`
- `secretsmanager:UntagResource`
- `secretsmanager:UpdateSecret`

## Security Groups

In order to restrict where the database can be accessed from, we need the
capability of creating security groups and security group rules.

- `ec2:AuthorizeSecurityGroupEgress`
- `ec2:AuthorizeSecurityGroupIngress`
- `ec2:CreateSecurityGroup`
- `ec2:DeleteSecurityGroup`
- `ec2:ModifySecurityGroupRules`
- `ec2:RevokeSecurityGroupEgress`
- `ec2:RevokeSecurityGroupIngress`

## RDS

Permissions required to create and manage the database server and a database
created within.

To prevent the capability of hosting the database in the default subnet group,
which uses subnnets outside of the VPC, we must also create and manage a
subnet group which spans the 3 availability zones.

- `rds:AddRoleToDBInstance`
- `rds:AddTagsToResource`
- `rds:CreateDBInstance`
- `rds:CreateDBParameterGroup`
- `rds:CreateDBSnapshot`
- `rds:CreateDBSubnetGroup`
- `rds:CreateOptionGroup`
- `rds:DeleteDBInstance`
- `rds:DeleteDBSubnetGroup`
- `rds:DeleteOptionGroup`
- `rds:Describe*`
- `rds:ListTagsForResource`
- `rds:ModifyDBInstance`
- `rds:ModifyDBSubnetGroup`
- `rds:RemoveTagsFromResource`

## STS

- `sts:AssumeRole`

The `sts:AssumeRole` permission must contain a condition granting permissions
to the role that will assume this one, as well as the `assumed-role`
credentials which in effect points back to this role.

```json
"Condition": {
    "ForAnyValue:StringLike": {
        "aws:PrincipalArn": [
            "arn:aws:sts::${AWS_ACCOUNT_ID}:role/crossplane-assume-role",
            "arn:aws:sts::${AWS_ACCOUNT_ID}:assumed-role/rds-crossplane-role*"
        ]
    }
}
```

## IAM

For certain capabilities we also require IAM permissions within the account
for the creation of `ServiceLinkedRole` and passing roles forward.

- `iam:CreateServiceLinkedRole`
- `iam:PassRole`

