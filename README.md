# [WIP] Using multiple network interfaces in OpenShift on AWS

### Prerequisites:
  
  - OpenShift Container Platform 4.3 or later (IPI deploy)
  - AWS CLI

### Know before you start...

  - In this example we chose "m4.large" Instance Type for our Worker Nodes, and its maximun number of network interfaces is 2 (two). Choose another Instance Type for your machines if you need more interfaces. See: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
  
  
### AWS CLI setup

  An example to setup your AWS CLI config

    export AWSKEY=<YOURACCESSKEY>
    export AWSSECRETKEY=<YOURSECRETKEY>
    export REGION=us-east-2

    mkdir $HOME/.aws
    cat << EOF >>  $HOME/.aws/credentials
    [default]
    aws_access_key_id = ${AWSKEY}
    aws_secret_access_key = ${AWSSECRETKEY}
    region = $REGION
    EOF
    
  Test your connection:
  
    $ aws sts get-caller-identity
    
  Sample output...
   
    {
        "Account": "588831877095",
        "UserId": "AIDALPAI4LLPASBJVUN2U",
        "Arn": "arn:aws:iam::588831877095:user/laparici@redhat.com-9c2a"
    }
  

## How to add new Interfaces to your Machines

### First, take a look around...

  1. View Instances info

    $ aws ec2 describe-instances --region=us-east-2 --output table
    
    
    We are gonna focus on machines inside Availability Zone "us-east-2a"
    
    $ aws ec2 describe-instances --filters "Name=availability-zone,Values=us-east-2a" \
    --query "Reservations[].Instances[].{InstaneId:InstanceId}"

  2. Get Instance's NetworkInterface details

    $ aws ec2 describe-instances --instance-ids i-0aa3b767615340d56
  
    $ aws ec2 describe-instances --instance-ids i-0aa3b767615340d56 --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'

    [
        [
            {
                "PrivateDnsName": "ip-10-0-133-56.us-east-2.compute.internal",
                "Primary": true,
                "PrivateIpAddress": "10.0.133.56"
            }
        ]
    ]

  Result: Only one NetworkInterface in this Instance, with IP 10.0.133.56



  3. (Optional) You can just add a new Private IP to an existent interface

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

  Notice that (only in this particular case) you'll have to config this new secondary IP on Node's Operating System. In this case RHCOS.


  4. Add new CIDR block to VPC
  
    Our new CIDR will be 10.2.0.0/16
  
  Create the new CIDR block:
  
    $ aws ec2 associate-vpc-cidr-block --vpc-id vpc-0b8fe5ab4a3d2095b --cidr-block 10.2.0.0/16
        
 See the results:
 
    $ aws ec2 describe-vpcs --vpc-id vpc-0b8fe5ab4a3d2095b  --query Vpcs[].CidrBlockAssociationSet
    [
        [
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
        ]
    ]


  5. Create new Subnet
  
    So lets create the subnet 10.2.30.0/24 (that's inside our new CIDR block 10.2.0.0/16)
  
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

  6. Create new network interface
  
  For this step you'll need Subnet Id from previous step
  
    $ aws ec2 create-network-interface --subnet-id subnet-0e01befdff050008a --description "my network interface"
    
    {
        "NetworkInterface": {
            "Status": "pending",
            "MacAddress": "02:38:93:ff:10:60",
            "SourceDestCheck": true,
            "AvailabilityZone": "us-east-2a",
            "Description": "my network interface",
            "NetworkInterfaceId": "eni-0a4803d1711b59281",
            "PrivateIpAddresses": [
                {
                    "PrivateDnsName": "ip-10-2-30-155.us-east-2.compute.internal",
                    "Primary": true,
                    "PrivateIpAddress": "10.2.30.155"
                }
            ],
            "RequesterManaged": false,
            "PrivateDnsName": "ip-10-2-30-155.us-east-2.compute.internal",
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
            "PrivateIpAddress": "10.2.30.155"
        }
    }
    
    
  7. Attach new Network Interface to VM Instance

    $ aws ec2 attach-network-interface --network-interface-id eni-0a4803d1711b59281 --instance-id i-0aa3b767615340d56 --device-index 1
    
    {
        "AttachmentId": "eni-attach-04c452afed7462b8d"
    }

    See the results, new interface attached to instance, with IP 10.2.30.155:
    
    $ aws ec2 describe-instances --instance-ids i-0aa3b767615340d56 --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'
    
    [
        [
            {
                "PrivateDnsName": "ip-10-0-133-56.us-east-2.compute.internal",
                "Primary": true,
                "PrivateIpAddress": "10.0.133.56"
            }
        ],
        [
            {
                "PrivateDnsName": "ip-10-2-30-155.us-east-2.compute.internal",
                "Primary": true,
                "PrivateIpAddress": "10.2.30.155"
            }
        ]
    ]

  8. Show me my new NIC
  
    $ oc debug node/ip-10-0-133-56.us-east-2.compute.internal
    Starting pod/ip-10-0-133-56us-east-2computeinternal-debug ...
    To use host binaries, run `chroot /host`
    Pod IP: 10.0.133.56
    If you don't see a command prompt, try pressing enter.
    sh-4.2# ip r
    default via 10.0.128.1 dev ens3 proto dhcp metric 100
    default via 10.2.30.1 dev ens4 proto dhcp metric 101
    10.0.128.0/19 dev ens3 proto kernel scope link src 10.0.133.56 metric 100
    10.2.30.0/24 dev ens4 proto kernel scope link src 10.2.30.155 metric 101
    10.128.0.0/14 dev tun0 scope link
    172.30.0.0/16 dev tun0    

  Done!, that's it!
  
  Now you can continue to add more interfaces to other Nodes




## Attach new network interfaces to the rest of the Nodes
  
  You can continue adding new network interfaces to the other Nodes in the OCP Cluster
  

#### Create new interface 

    $ aws ec2 create-network-interface --subnet-id subnet-0e01befdff050008a --description "my network interface"
    {
        "NetworkInterface": {
            "Status": "pending",
            "MacAddress": "02:bd:b5:b7:82:b8",
            "SourceDestCheck": true,
            "AvailabilityZone": "us-east-2a",
            "Description": "my network interface",
            "NetworkInterfaceId": "eni-0c9e3d7992faaa308",
            "PrivateIpAddresses": [
                {
                    "PrivateDnsName": "ip-10-2-30-170.us-east-2.compute.internal",
                    "Primary": true,
                    "PrivateIpAddress": "10.2.30.170"
                }
            ],
            "RequesterManaged": false,
            "PrivateDnsName": "ip-10-2-30-170.us-east-2.compute.internal",
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
            "PrivateIpAddress": "10.2.30.170"
        }
    }


#### List your Nodes
    
    $ oc get nodes
    NAME                                         STATUS   ROLES    AGE    VERSION
    ip-10-0-133-117.us-east-2.compute.internal   Ready    worker   3h13m   v1.18.3+47c0e71
    ip-10-0-133-56.us-east-2.compute.internal    Ready    worker   6m27s   v1.18.3+47c0e71
    ip-10-0-150-203.us-east-2.compute.internal   Ready    master   3h24m   v1.18.3+47c0e71
    ip-10-0-153-175.us-east-2.compute.internal   Ready    worker   6m27s   v1.18.3+47c0e71
    ip-10-0-171-48.us-east-2.compute.internal    Ready    worker   3h13m   v1.18.3+47c0e71
    ip-10-0-177-26.us-east-2.compute.internal    Ready    master   3h24m   v1.18.3+47c0e71
    ip-10-0-212-213.us-east-2.compute.internal   Ready    master   3h24m   v1.18.3+47c0e71
    ip-10-0-219-114.us-east-2.compute.internal   Ready    worker   3h13m   v1.18.3+47c0e71

#### Identify the Node's Instance Id in AWS
    
    $ aws ec2 describe-instances --filters "Name=availability-zone,Values=us-east-2a" \
    --query "Reservations[].Instances[].{InstaneId:InstanceId,PrivateDnsName:PrivateDnsName}"
    
    [
        {
            "InstaneId": "i-0aa3b767615340d56",
            "PrivateDnsName": "ip-10-0-133-56.us-east-2.compute.internal"
        },
        {
            "InstaneId": "i-0c07ca328279340cf",
            "PrivateDnsName": "ip-10-0-133-117.us-east-2.compute.internal"
        },
        {
            "InstaneId": "i-0ee574517afd2739c",
            "PrivateDnsName": "ip-10-0-153-175.us-east-2.compute.internal"
        },
        {
            "InstaneId": "i-0103cd5c3d3e069bb",
            "PrivateDnsName": "ip-10-0-150-203.us-east-2.compute.internal"
        }
    ]

#### Attach the new interface

    aws ec2 attach-network-interface --network-interface-id eni-0c9e3d7992faaa308 --instance-id i-0ee574517afd2739c --device-index 1
    {
        "AttachmentId": "eni-attach-01487f52c832c3fcb"
    }
    
#### The new NIC inside the Node

  And there it is! New "ens4" interface with IP 10.2.30.170

    $ oc debug node/ip-10-0-153-175.us-east-2.compute.internal
    Starting pod/ip-10-0-153-175us-east-2computeinternal-debug ...
    To use host binaries, run `chroot /host`
    Pod IP: 10.0.153.175
    If you don't see a command prompt, try pressing enter.
    sh-4.2# ip r
    default via 10.0.128.1 dev ens3 proto dhcp metric 100
    default via 10.2.30.1 dev ens4 proto dhcp metric 101
    10.0.128.0/19 dev ens3 proto kernel scope link src 10.0.153.175 metric 100
    10.2.30.0/24 dev ens4 proto kernel scope link src 10.2.30.170 metric 101
    10.128.0.0/14 dev tun0 scope link
    172.30.0.0/16 dev tun0
    sh-4.2#
    
  
## Pods with additional interface

  #### Create new Project/Namespace

    $ oc new-project test-new-nic
    
  #### Add Node Selector to nodes and Namespace
  
  For Namespace add this annotation:
    
    ...
    annotations:
      openshift.io/node-selector: "test-new-nic=true"
    ...
    
    
  And label the nodes that have the new interface "ens4"
  
  
    $ oc label node ip-10-0-153-175.us-east-2.compute.internal ip-10-0-133-56.us-east-2.compute.internal test-new-nic=true
    
    
  #### Config Namespace Additional Network
  
  Note that in AWS environments, macvlan traffic might be filtered and, therefore, might not reach the desired destination. Use ipval type instead, for example
  
    $ oc edit networks.operators
    
    ...
    spec:
      additionalNetworks:
        - name: test-new-nic
          namespace: test-new-nic
          rawCNIConfig: '{ "cniVersion": "0.3.1", "type": "macvlan", "capabilities": { "ips": true }, "master": "ens4", "mode": "bridge", "ipam": { "type": "static" } }'
          type: Raw
      clusterNetwork:
    ...

  Switch to Project test-new-nic and list your NetworkAttachmentDefinitions:

    oc get network-attachment-definitions
    NAME           AGE
    test-new-nic   79s


oc create serviceaccount -n test-new-nic privilegeduser
serviceaccount/privilegeduser created

 oc adm policy add-scc-to-user privileged -n test-new-nic -z privilegeduser
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "privilegeduser"



## (WIP) AWS CNI

    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: aws
    spec:
      config: '{"cniVersion":"0.3.0","name":"aws-cni","type":"aws-cni","vethPrefix":"eni"}'



