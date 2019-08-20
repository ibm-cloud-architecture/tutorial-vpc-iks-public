# Use GUI to Connect an App deployed within VPC to an IKS deployment outside VPC

## Prerequisites


1. Up and running IKS cluster with helm installed/initialized.
2. Strongswan deployed to the IKS as per these [instructions](./IKS-strongswan.md)
3. Identify the Strongswan pre-shared key
   Example: presharedkey=<value of your pre-shared key>
4. Identify the Strongswan Gateway IP
   Example: gatewayip=169.63.235.152

## Login to IBM Cloud

Browse to https://cloud.ibm.com/login and login.

<kbd>![Log in](files/images/00-login.png)</kbd>

Select VPC Infrastructure from the hamberger menu in the upper left corner.

<kbd>![VPC Infrastructure](files/images/01-menu-vpc.png)</kbd>

## Deploy VPC Infrastructure

### Create an SSH Key

1. An SSH key is required when creating a VPC instance. From the Compute menu slect "SSH keys" 
<kbd>![SSH Key](files/images/10-sshkey-overview.png)</kbd>
1. Click the "Add SSH key" button. In the modal dialog provide a name & public key and click the "Add SSH key" button.
<kbd>![Add SSH Key](files/images/11-sshkey-create.png)</kbd>
3. Confirm the ssh key was added.
<kbd>![Confirm SSH Key](files/images/12-sshkey-confirm.png)</kbd>

### Resource Group, Region and Zone

For this scenario use group `priyank` instead of default, and for location use Dallas with zone Dallas 1.

### Create VPC

1. From the `Network` menu in the navigation bar select `VPC and Subnets` & click the "New virtual private cloud" button.
<kbd>![VPC Overview](files/images/20-vpc-overview.png)</kbd>
2. Click the "New virtual private cloud" button. Next fill out the form and hit the "Create virtual private cloud" button
<kbd>![VPC Create](files/images/21-vpc-create.png)</kbd>
3. Confirm the newly created VPC is available
<kbd>![Confirm VPC Availablity](files/images/22-vpc-confirm.png)</kbd>   
4. Check the status of the subnet. Initially it will be set to pending.
<kbd>![Pending Subnet Status](files/images/23-subnet-pending.png)</kbd>
5. Confirm the status of the subnet is `available`
<kbd>![Confirm Subnet Status](files/images/24-subnet-available.png)</kbd>

## Create the IKE & IPSec Policy
Internet Key Exchange (IKE) is a key management protocol that is used to authenticate IPsec peers,
negotiate and distribute IPsec encryption keys, and to automatically establish IPsec security associations
(SAs).
1. From the `Network` menu in the navigation bar select `VPNs` & click the "IKE policies" tab.
<kbd>![IKE Overview](files/images/30-vpn-ike-overview.png)</kbd>
2. Click the "New IKE policy" button. Next fill out the form and hit the "Confirm" button
<kbd>![IKE Create](files/images/31-vpn-ike-create.png)</kbd>
3. Confirm the IKE policy is created and available
<kbd>![IKE Create](files/images/32-vpn-ike-confirm.png)</kbd>
4. Next toggle to the "IPsec policies" tab.
<kbd>![IKE Create](files/images/33-vpn-ipsec-overview.png)</kbd>
5. Click the "New IPsec policy" button. Next fill out the form and hit the "Confirm" button
<kbd>![IKE Create](files/images/34-vpn-ipsec-create.png)</kbd>
6. Confirm the IPsec policy is created and available
<kbd>![IKE Create](files/images/35-vpn-ipsec-confirm.png)</kbd>

## Create the VPN Gateway
1. From the `Network` menu in the navigation bar select `VPNs` & click the "VPN Gateways" tab.
   <kbd>![IKE Overview](files/images/40-vpn-gateway-overview.png)</kbd>
2. Click the "New VPN gateway" button. Next fill out the form and hit the "Confirm" button.
    <kbd>![IKE Overview](files/images/41-vpn-gateway-conn-create.png)</kbd>
3. Confirm the VPN gateway is created and available
   <kbd>![IKE Overview](files/images/42-vpn-gateway-confirm.png)</kbd>
4. Open the VPN Gateway details by Clicking on the name, Next click on "View all connections" link.
    <kbd>![IKE Overview](files/images/43-vpn-gateway-details.png)</kbd>
5. Confirm the VPN gateway connection is created and active
   <kbd>![IKE Overview](files/images/44-vpn-gateway-conn-confirm.png)</kbd>

## (Optional) Configure the DNS on the resources in VPC to access the Kubernetes deployments
Any deployments in the VPC should now be able to access the pods and services deployed in the kubernetes cluster. No configuration is needed if the pod IP or cluster ip is used to access the resources. However, if the kubernetes resources are to be accessed by DNS lookup, the dns settings for the vsi or other resources in the VPC needs to be updated and set the kubernetes nameserver to `172.21.0.10` . 

## Error Scenarios

List error scenarios.

## Documentation Provided

This section will provide links to all required documentation for this use case. 

<links to documents>
