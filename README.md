# openshift-aws-multus


## Show AWS instances

  aws ec2 describe-instances --region=us-east-2 --output table

## Describe AWS specific Instance

  aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb 
  
  aws ec2 describe-instances --instance-ids i-0103cd5c3d3e069bb --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses'

## Add new Private IP to AWS instance

  aws ec2 assign-private-ip-addresses --network-interface-id eni-0d61b77c3ca6ab811 --secondary-private-ip-address-count 1
  
