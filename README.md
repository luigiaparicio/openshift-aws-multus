# openshift-aws-multus


## Show AWS instances

    aws ec2 describe-instances --region=us-east-2 --output table

## Describe AWS specific Instance

    aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb 
  
    aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'

## Add new Private IP to AWS instance

    aws ec2 assign-private-ip-addresses --network-interface-id eni-0d61b77c3ca6ab811 --secondary-private-ip-address-count 1
  
    aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'
    [
        [
            {
                "PrivateDnsName": "ip-10-0-150-203.us-east-2.compute.internal",
                "Primary": true,
                "PrivateIpAddress": "10.0.150.203"
           },
            {
               "PrivateDnsName": "ip-10-0-152-75.us-east-2.compute.internal",
               "Primary": false,
               "PrivateIpAddress": "10.0.152.75"
            }
        ]
    ]
  
  ## Add new CIDR block to VPC
    aws ec2 associate-vpc-cidr-block --vpc-id vpc-0b8fe5ab4a3d2095b --cidr-block 10.2.0.0/16
        
    $ aws ec2 describe-vpcs --vpc-id vpc-0b8fe5ab4a3d2095b
    {
        "Vpcs": [
            {
                "VpcId": "vpc-0b8fe5ab4a3d2095b",
                "InstanceTenancy": "default",
                "Tags": [
                    {
                        "Value": "cluster-9c2a-jlt54-vpc",
                        "Key": "Name"
                    },
                    {
                        "Value": "owned",
                        "Key": "kubernetes.io/cluster/cluster-9c2a-jlt54"
                    }
                ],
                "CidrBlockAssociationSet": [
                    {
                        "AssociationId": "vpc-cidr-assoc-07fbd8750573fcf70",
                        "CidrBlock": "10.0.0.0/16",
                        "CidrBlockState": {
                             "State": "associated"
                        }
                    },
                    {
                        "AssociationId": "vpc-cidr-assoc-0f349b1a1e1eaa329",
                        "CidrBlock": "10.2.0.0/16",
                        "CidrBlockState": {
                             "State": "associated"
                        }
                    }
                ],
                "State": "available",
                "DhcpOptionsId": "dopt-0b85ee5967b6e605d",
                "OwnerId": "588831808095",
                "CidrBlock": "10.0.0.0/16",
                "IsDefault": false
            }
        ]
    }
  
  
  
