# Use CLI to Connect an App deployed within VPC to an IKS deployment outside VPC

## Prerequisites

1. Install the [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started)
2. Have access to a public SSH key.
3. Install the infrastructure-service plugin.  
   `$ ibmcloud plugin install infrastructure-service`
4. Up and running IKS cluster with helm installed/initialized.
5. Strongswan deployed to the IKS as per these [instructions](./IKS-strongswan.md)
6. Create a environment variable for the Strongswan pre-shared key
   Example: `$ export PRESHARED_KEY=teiyoo1shie5eGh1ooyeNg0chooShaiV`
7. Create a environment variable for the Strongswan Gateway IP
   Example: `export GATEWAY_IP=169.63.235.152`

## Login to IBM Cloud

For a federated account use single sign on:  
   `$ ibmcloud login -sso`  
Otherwise use the default login:  
   `$ ibmcloud login`  
If you have an API Key, use --apikey:  
   `$ ibmcloud login --apikey [your API Key]`  

## Set Resource Group, Region and Zone

New VPC resources will be assigned the account's default Resource Group.  Use the ibmcloud target command to select the desired group and region for the VPC.  In our case we want to use group VPC1 instead of default, and locate the VPC in the us-south region. 

```
$ ic resource groups
Retrieving all resource groups under account Phillip Trent's Account as pltrent@us.ibm.com...
OK
Name      ID                                 Default Group   State   
default   00d24065a2ec44efb9de172e6d19b919   true            ACTIVE   
VPC1      04620a177bad4baf999ad5704eaae2d2   false           ACTIVE
priyank   4fe35a9ad2b5438291bd88dae7aa208d   false           ACTIVE  
```

```
$ ic target  -g priyank -r us-south
Targeted resource group priyank

Switched to region us-south

API endpoint:      https://api.ng.bluemix.net   
Region:            us-south   
User:              priyankn@ca.ibm.com   
Account:           Phillip Trent's Account (843f59bad5553123f46652e9c43f9e89) <-> 1691265   
Resource group:    priyank   
CF API endpoint:      
Org:                  
Space:        
```
Now that we are in us-south, let's find out what zones are available using the zones command. We are going to use us-south-1 for this example.

```
$ ic is zones us-south
Listing zones in region us-south under account Phillip Trent's Account as user priyankn@ca.ibm.com...
Name         Region     Status   
us-south-3   us-south   available   
us-south-1   us-south   available   
us-south-2   us-south   available 

$ export ZONE_NAME=us-south-1
```

## Create an SSH Key

An SSH key is required when creating a VPC instance. Copy the ssh public key you wish to use to vpc-key.pub and call the key-create command to load it to the VPC environment. Remember the key id for later use.

```
$ ic is key-create priyank-mac @/Users/priyanknarvekar/.ssh/id_rsa.pub
Creating key priyank-mac under account Phillip Trent's Account as user priyankn@ca.ibm.com...
                 
ID            636f6d70-0000-0001-0000-000000140a55   
Name          priyank-mac   
Type          rsa   
Length        2048   
FingerPrint   SHA256:7LMi9Nn8SHDCnFovCaMSv+IIE5KxR7EXSGTA5aqW3RE   
Key           ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCylLp6CbMlvb8TQnAbmoUkvlq/90/bo4H1B7cjWIBdn44qrywvHYayVoHygZfMAjfcFdtvtJwGjGqg502z7eRg0yzh5uG4iIvF6CawL0N4QSQNkiJl5OW0RIswcQ8U+9wj1K0C3A2l7JkNi+Jc+A86FtQKFkXcjO/702DGR+bYFq2FdwajLxmCAd7zWrVGZXi8/5G1yrYDKcEdxVpgR/DAMNieRzcbZbuA62jDn7zf9Iab6CAM8Gd8AmdisoO4Pg0dByUkg6I6XQ2QhkUrmG9C6guNTM9xchmMPVFYY7QwLpgR/PLd54/jtNESNkn0MiYJW6uTGmGyZOgFsoaIBs6j   
                 
Created       1 second ago 

$ export KEY_ID=636f6d70-0000-0001-0000-000000140a55
```

## Create VPC
Create a VPC named iks-integration.

```
$ ic is vpc-create iks-integration
Creating vpc iks-integration in resource group priyank under account Phillip Trent's Account as user priyankn@ca.ibm.com...
                            
ID                       3c19c715-0a26-466f-a6c7-f4c8c98e5e77   
Name                     iks-integration   
Default                  no   
Default Network ACL      allow-all-network-acl-3c19c715-0a26-466f-a6c7-f4c8c98e5e77(bb1e86fc-89a7-41cd-9c6b-772469e5a737)   
Default Security Group   limeade-freezing-blanching-backlash-mooing-anagram(2d364f0a-a870-42c3-a554-000001165547)   
Resource Group           (4fe35a9ad2b5438291bd88dae7aa208d)   
Created                  12 seconds ago   
Status                   available   

$ export VPC_ID=3c19c715-0a26-466f-a6c7-f4c8c98e5e77
```

## Create Address Prefix

Create address prefixes for 10.10.10.0/24

```
$ ic is vpc-address-prefix-create orange $VPC_ID $ZONE_NAME 10.10.10.0/24
Creating address prefix orange of vpc 3c19c715-0a26-466f-a6c7-f4c8c98e5e77 under account Phillip Trent's Account as user priyankn@ca.ibm.com...

ID            c905e5a2-6ec0-4eff-90cb-69ea81355a7e
Name          orange   
CIDR Block    10.10.10.0/24   
Zone          us-south-1   
Has Subnets   no   
Created       1 second ago 

$ export PREFIX_ID=c905e5a2-6ec0-4eff-90cb-69ea81355a7e
```

## Create the VPC Subnet

Create a VPC subnet in us-south-1 for ipv4-cidr-blocks 10.10.10.0/24.  
The initial status of a newly created subnet is set to pending.  
You must wait until the subnet staus is available before assiging any resources to it.

```
$ ic is subnet-create orange $VPC_ID us-south-1 --ipv4-cidr-block 10.10.10.0/24
Creating Subnet orange under account Phillip Trent's Account as user priyankn@ca.ibm.com...
                    
ID               22d8114e-114d-4172-ba4b-8840adc35cc5   
Name             orange   
IPv*             ipv4   
IPv4 CIDR        10.10.10.0/24   
IPv6 CIDR        -   
Addr available   248   
Addr Total       256   
ACL              allow-all-network-acl-3c19c715-0a26-466f-a6c7-f4c8c98e5e77(bb1e86fc-89a7-41cd-9c6b-772469e5a737)   
Gateway          -   
Created          1 second ago  
Status           pending   
Zone             us-south-1   
VPC              iks-integration(3c19c715-0a26-466f-a6c7-f4c8c98e5e77) 

$ export SUBNET_ID=22d8114e-114d-4172-ba4b-8840adc35cc5
```

## Create the IKE & IPSec Policy
Internet Key Exchange (IKE) is a key management protocol that is used to authenticate IPsec peers,
negotiate and distribute IPsec encryption keys, and to automatically establish IPsec security associations
(SAs).

```
$ ic is ike-policy-create iks-ike2-policy sha1 2 aes256 2 --key-lifetime 86400
Creating IKE policy iks-ike2-policy under account Phillip Trent's Account as user priyankn@ca.ibm.com...

ID                         40eefd4c-7bcd-4d37-b395-f32e3deca39c
Name                       iks-ike2-policy
Authentication Algorithm   sha1
Diffie-Hellman Group       2
Encryption Algorithm       aes256
IKE Protocol Version       2
Key Lifetime(seconds)      86400
VPN Connections            -
IKE Negotiation Mode       main

$ ic is ipsec-policy-create iks-esp-policy sha1 aes256 disabled --key-lifetime 3600
Creating IPsec policy iks-esp-policy1 under account Phillip Trent's Account as user priyankn@ca.ibm.com...
                              
ID                         81cfbaa2-c398-4926-a7b0-82069027bc85   
Name                       iks-esp-policy   
Authentication Algorithm   sha1   
Encryption Algorithm       aes256   
Key Lifetime(seconds)      3600   
Perfect Forward Secrecy    disabled   
VPN Connections            -   
Encapsulation Mode         tunnel   
Transform Protocol         esp   

$ export IKE_POLICY_ID=40eefd4c-7bcd-4d37-b395-f32e3deca39c
$ export IPSEC_POLICY_ID=81cfbaa2-c398-4926-a7b0-82069027bc85
```

## Create the VPN Gateway

```
$ ic is vpn-gateway-create iks-vpn-gw $SUBNET_ID
Creating vpn gateway iks-vpn-gw under account Phillip Trent's Account as user priyankn@ca.ibm.com...
                     
ID                4cbc689a-572a-48b4-a641-3f40f733926f   
Name              iks-vpn-gw   
Subnet            22d8114e-114d-4172-ba4b-8840adc35cc5   
Public IP         169.61.244.31   
Resource Group    4fe35a9ad2b5438291bd88dae7aa208d   
VPN Connections   2c808857-4aea-44a3-8e5a-3fa2a5574c54   
Created           1 day ago   
Status            available  

$ export VPN_GW_ID=4cbc689a-572a-48b4-a641-3f40f733926f
```

## Create the VPN Gateway Connection

```
$ ibmcloud is vpn-gateway-connection-create \
    iks-conn $VPN_GW_ID $GATEWAY_IP $PRESHARED_KEY \      
    --ike-policy $IKE_POLICY_ID \
    --ipsec-policy $IPSEC_POLICY_ID \
    --local-cidrs 10.10.10.0/24 \
    --peer-cidrs 172.30.0.0/16 \
    --peer-cidrs 172.21.0.0/16

Creating connection iks-conn of vpn gateway 4cbc689a-572a-48b4-a641-3f40f733926f under account Phillip Trent's Account as user priyankn@ca.ibm.com...

ID                           2c808857-4aea-44a3-8e5a-3fa2a5574c54
Name                         iks-conn
Admin State up               yes
IKE Policy                   40eefd4c--4d37-b395-f32e3deca39c
IPsec Policy                 81cfbaa2-c397bcd8-4926-a7b0-82069027bc85
Peer IP Address              169.46.87.109
Preshared Key                teiyoo1shie5eGh1ooyeNg0chooShaiV
Authentication Mode          psk
Created                      4 days ago
Route Mode                   policy
Status                       up
Dead Peer Detection Action   none
Local CIDRs                  10.10.10.0/24
Peer CIDRs                   172.21.0.0/16,172.30.0.0/16

```

## Start the vpn connection

```
ic is vpn-cnu $VPN_GW_ID $CONN_ID --admin-state-up true
```

## (Optional) Configure the DNS on the resources in VPC to access the Kubernetes deployments
Any deployments in the VPC should now be able to access the pods and services deployed in the kubernetes cluster. No configuration is needed if the pod IP or cluster ip is used to access the resources. However, if the kubernetes resources are to be accessed by DNS lookup, the dns settings for the vsi or other resources in the VPC needs to be updated and set the kubernetes nameserver to `172.21.0.10` . 

## Error Scenarios

List error scenarios.

## Documentation Provided

This section will provide links to all required documentation for this use case. 

<links to documents>