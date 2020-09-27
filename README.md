# openshift-aws-multus


## 1) Show AWS instances

    $ aws ec2 describe-instances --region=us-east-2 --output table

## 2) Describe AWS specific Instance

    $ aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb 
  
    $ aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'

## 3) Add new Private IP to AWS instance

    $ aws ec2 assign-private-ip-addresses --network-interface-id eni-0d61b77c3ca6ab811 --secondary-private-ip-address-count 1
  
    $ aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'
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
  
  ## 4) Add new CIDR block to VPC
    $ aws ec2 associate-vpc-cidr-block --vpc-id vpc-0b8fe5ab4a3d2095b --cidr-block 10.2.0.0/16
        
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
  
  
  ## 5) Create new Subnet
  
  Pick the right Availability Zone name (see step 2 above)
  
      $ aws ec2 create-subnet --vpc-id vpc-0b8fe5ab4a3d2095b --cidr-block 10.2.30.0/24 --availability-zone us-east-2a

    {
        "Subnet": {
            "MapPublicIpOnLaunch": false,
            "AvailabilityZoneId": "use2-az1",
            "AvailableIpAddressCount": 251,
            "DefaultForAz": false,
            "SubnetArn": "arn:aws:ec2:us-east-2:588831808095:subnet/subnet-0e01befdff050008a",
            "Ipv6CidrBlockAssociationSet": [],
            "VpcId": "vpc-0b8fe5ab4a3d2095b",
            "State": "available",
            "AvailabilityZone": "us-east-2a",
            "SubnetId": "subnet-0e01befdff050008a",
            "OwnerId": "588831808095",
            "CidrBlock": "10.2.30.0/24",
            "AssignIpv6AddressOnCreation": false
        }
    }

  ## 6) Create new network interface
  
  Pick Subnet Id from previous step
  
    $ aws ec2 create-network-interface --subnet-id subnet-0e01befdff050008a --description "my network interface"

    {
        "NetworkInterface": {
            "Status": "pending",
            "MacAddress": "02:fb:7b:dd:a2:92",
            "SourceDestCheck": true,
            "AvailabilityZone": "us-east-2a",
            "Description": "my network interface",
            "NetworkInterfaceId": "eni-0db2ba590ecd24f56",
            "PrivateIpAddresses": [
                {
                    "PrivateDnsName": "ip-10-2-30-219.us-east-2.compute.internal",
                    "Primary": true,
                    "PrivateIpAddress": "10.2.30.219"
                }
            ],
            "RequesterManaged": false,
            "PrivateDnsName": "ip-10-2-30-219.us-east-2.compute.internal",
            "VpcId": "vpc-0b8fe5ab4a3d2095b",
            "InterfaceType": "interface",
            "RequesterId": "AIDAYSGI4LJPSSBJVUN2U",
            "Groups": [
                {
                    "GroupName": "default",
                    "GroupId": "sg-0eea8477dae0eeda8"
                }
            ],
            "Ipv6Addresses": [],
            "OwnerId": "588831808095",
            "SubnetId": "subnet-0e01befdff050008a",
            "TagSet": [],
            "PrivateIpAddress": "10.2.30.219"
        }
    }
    
    
## Attach new Network Interface to VM Instance

    $ aws ec2 attach-network-interface --network-interface-id eni-0db2ba590ecd24f56 --instance-id i-0103cd5c3d3e069bb --device-index 2
    {
    "AttachmentId": "eni-attach-08ae3e6ade89d01c8"
    }
    
    $ aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'
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
        ],
        [
            {
                "PrivateDnsName": "ip-10-2-30-219.us-east-2.compute.internal",
                "Primary": true,
                "PrivateIpAddress": "10.2.30.219"
            }
        ]
    ]
    
