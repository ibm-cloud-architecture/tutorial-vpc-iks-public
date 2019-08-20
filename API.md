# Use API to Connect an App deployed within VPC to an IKS deployment outside VPC

## Prerequisites
1. [Generate an IBM Cloud API Key](https://cloud.ibm.com/docs/iam?topic=iam-userapikey).
   Once generated, create an environment variable labeled apikey. 
   Example: apikey=value of your apikey
2. Have access to a public SSH key.
3. curl & jq command
4. Up and running IKS cluster with helm installed/initialized.
5. Strongswan deployed to the IKS as per these [instructions](./IKS-strongswan.md)
6. Create a environment variable for the Strongswan pre-shared key
   Example: presharedkey=value of your pre-shared key
7. Create a environment variable for the Strongswan Gateway IP
   Example: gatewayip=169.63.235.152
   
## Generate an IAM Bearer Token
[Documentation](https://cloud.ibm.com/docs/iam?topic=iam-iamtoken_from_apikey)\
Issue the following to generate a Bearer token and store it in the `iam_token` env variable:

```
export iam_token=$(curl -s -k -X POST \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --header "Accept: application/json" \
  --data-urlencode "grant_type=urn:ibm:params:oauth:grant-type:apikey" \
  --data-urlencode "apikey=$apikey" \
  "https://iam.cloud.ibm.com/identity/token" | jq -r '.access_token') &&  \
  [ "$iam_token" != "null" ] && \
  echo "iam_token=$iam_token" || \
  >&2 echo "Failed to initialize IAM Token"
```

## Set the Resource Group to deploy the vpc resources in

If the name of the resource group is already known, issue the following export command identifying the correct resource group name(instead of `priyank`):
```
export resource_group=$(curl -s -k -X GET \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $iam_token" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  https://resource-manager.bluemix.net/v1/resource_groups | \
  jq -r -c '.resources | .[] | select(.name=="priyank") | .id') &&  \
  [ "$resource_group" != "null" ] && [ "$resource_group" != "" ] && \
  echo "resource_group=$resource_group" || \
  >&2 echo "Failed to initialize Resource Group id"

```
Alternatively all the resource groups in an account can be listed using the following curl command:
```
curl -s -k -X GET -H "Accept: application/json" \
   -H "Authorization: Bearer $iam_token" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  https://resource-manager.bluemix.net/v1/resource_groups
```

Decide which resource group you wish to use for the VPC and store it in an environment variable called `resource_group`:\
  `resource_group=4fe35a9ad2b5438291bd88dae7aa208d`


## Create a SSH Key
To create a SSH Key use replacing the public_key with appropriate public key data: 
```
export key_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/keys?version=2019-01-01" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.1" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"name\":\"priyank-mac\",\"public_key\":\"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCylLp6CbMlvb8TQnAbmoUkvlq/90/bo4H1B7cjWIBdn44qrywvHYayVoHygZfMAjfcFdtvtJwGjGqg502z7eRg0yzh5uG4iIvF6CawL0N4QSQNkiJl5OW0RIswcQ8U+9wj1K0C3A2l7JkNi+Jc+A86FtQKFkXcjO/702DGR+bYFq2FdwajLxmCAd7zWrVGZXi8/5G1yrYDKcEdxVpgR/DAMNieRzcbZbuA62jDn7zf9Iab6CAM8Gd8AmdisoO4Pg0dByUkg6I6XQ2QhkUrmG9C6guNTM9xchmMPVFYY7QwLpgR/PLd54/jtNESNkn0MiYJW6uTGmGyZOgFsoaIBs6j\",\"type\":\"rsa\"}" | \
  jq -r -c '.id') &&  \
  [ "$key_id" != "null" ] && [ "$key_id" != "" ] && \
  echo "key_id=$key_id" || \
  >&2 echo "Failed to initialize SSH Key id"
```

## Create/Provision a VPC

```
export vpc_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/vpcs" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.0" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"name\":\"iks-vpc\",\"resource_group\":{\"id\":\"$resource_group\"}}" | \
  jq -r ".id") &&  \
  [ "$vpc_id" != "null" ] && [ "$vpc_id" != "" ] && \
  echo "vpc_id=$vpc_id" || \
  >&2 echo "Failed to create VPC"
```

## Capture the VPC's Security Group
```
export default_security_group=$(curl -s -X GET \
  "https://us-south.iaas.cloud.ibm.com/v1/vpcs/$vpc_id" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.0" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  | jq -r ".default_security_group | .id") && \
  [ "$default_security_group" != "null" ] && \
  [ "$default_security_group" != "" ] && \
  echo "default_security_group=$default_security_group" || \
  >&2 echo "Failed to capture default_security_group"
```

## Create a subnet in your VPC
```
export subnet_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/subnets" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.0" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"zone\":{\"name\":\"us-south-1\"},\"ip_version\":\"ipv4\",\"name\":\"orange\",\"ipv4_cidr_block\":\"10.240.0.0/24\",\"vpc\":{\"id\":\"$vpc_id\"}}" \
  | jq -r ".id") &&  \
  [ "$subnet_id" != "null" ] && [ "$subnet_id" != "" ] && \
  echo "subnet_id=$subnet_id" || \
  >&2 echo "Failed to create Subnet"
```

## Create the IKE Policy
Internet Key Exchange (IKE) is a key management protocol that is used to authenticate IPsec peers,
negotiate and distribute IPsec encryption keys, and to automatically establish IPsec security associations
(SAs).

```
export ike_policy_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/ike_policies?version=2019-01-01" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.1" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"name\":\"iks-ike2-policy\",\"authentication_algorithm\":\"sha1\",\"encryption_algorithm\":\"aes256\",\"key_lifetime\":86400,\"ike_version\":2,\"dh_group\":2,\"resource_group\":{\"id\":\"$resource_group\"}}" \
  | jq -r ".id") &&  \
  [ "$ike_policy_id" != "null" ] && [ "$ike_policy_id" != "" ] && \
  echo "ike_policy_id=$ike_policy_id" || \
  >&2 echo "Failed to create IKE Policy"
```

## Create the IPSec Policy
```
export ipsec_policy_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/ipsec_policies?version=2019-01-01" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.1" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"name\":\"iks-esp-policy\",\"authentication_algorithm\":\"sha1\",\"encryption_algorithm\":\"aes256\",\"key_lifetime\":3600,\"pfs\":\"disabled\",\"resource_group\":{\"id\":\"$resource_group\"}}" \
  | jq -r ".id") &&  \
  [ "$ipsec_policy_id" != "null" ] && [ "$ipsec_policy_id" != "" ] && \
  echo "ipsec_policy_id=$ipsec_policy_id" || \
  >&2 echo "Failed to create IPSec Policy"

```

## Create the VPN Gateway
```
export vpn_gateway_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/vpn_gateways?version=2019-01-01" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.1" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"name\":\"iks-vpn-gw\",\"subnet\":{\"id\":\"$subnet_id\"},\"resource_group\":{\"id\":\"$resource_group\"}}" \
  | jq -r ".id") &&  \
  [ "$vpn_gateway_id" != "null" ] && [ "$vpn_gateway_id" != "" ] && \
  echo "vpn_gateway_id=$vpn_gateway_id" || \
  >&2 echo "Failed to create VPN gateway"
```


## Create the VPN Gateway Connection

```
export vpn_conn_id=$(curl -s -X POST \
  "https://us-south.iaas.cloud.ibm.com/v1/vpn_gateways/$vpn_gateway_id/connections?version=2019-01-01" \
  -H "Authorization: Bearer $iam_token" \
  -H "User-Agent: IBM_One_Cloud_IS_UI/2.4.1" \
  -H "Content-Type: application/json" \
  -H "Cache-Control: no-cache" \
  -H "accept: application/json" \
  -d "{\"local_cidrs\":[\"10.240.0.0/24\"],\"name\":\"iks-conn\",\"peer_address\":\"$gatewayip\",\"peer_cidrs\":[\"172.30.0.0/16\",\"172.21.0.0/16\"],\"psk\":\"$presharedkey\",\"dead_peer_detection\":{\"action\":\"none\",\"interval\":30,\"timeout\":150},\"ipsec_policy\":{\"id\":\"$ipsec_policy_id\"},\"ike_policy\":{\"id\":\"$ike_policy_id\"}}" \
  | jq -r ".id") &&  \
  [ "$vpn_conn_id" != "null" ] && [ "$vpn_conn_id" != "" ] && \
  echo "vpn_conn_id=$vpn_conn_id" || \
  >&2 echo "Failed to create VPN gateway connection"
```

## (Optional) Configure the DNS on the resources in VPC to access the Kubernetes deployments
Any deployments in the VPC should now be able to access the pods and services deployed in the kubernetes cluster. No configuration is needed if the pod IP or cluster ip is used to access the resources. However, if the kubernetes resources are to be accessed by DNS lookup, the dns settings for the vsi or other resources in the VPC needs to be updated and set the kubernetes nameserver to `172.21.0.10` . 

## Error Scenarios

List error scenarios.

## Documentation Provided

This section will provide links to all required documentation for this use case. 

<links to documents>
