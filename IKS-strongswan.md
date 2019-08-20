## Deploy StrongSwan helm chart on IKS

Create a configuration template for strongswan deployment
```
helm inspect values ibm/strongswan > config.yaml
```
Customize the config.yaml for setting the following properties
```
enableServiceSourceIP: true
connectUsingLoadBalancerIP: "true"

ipsec:
  keyexchange: ikev2
  esp: "aes256-sha1-modp1024"
  ike: "aes256-sha1-modp1024"
  auto: add
  closeaction: auto
  dpdaction: restart
  ikelifetime: 86400
  keyingtries: "%forever"
  lifetime: "1h"
  margintime: "9m"

local:
  subnet: "172.30.0.0/16,172.21.0.0/16"
  id: ""

remote:
  subnet: "10.240.0.0/24"       # configure this to the Subnet range of the VPC
  id: ""

preshared:
  secret: "teiyoo1shie5eGh1ooyeNg0chooShaiV"
```

Install the strongswan helm chart using the config.yaml
```
helm install -f config.yaml --name=vpn-vpc ibm/strongswan
```

Finally capture the Public IP of the Strongswan deployment by listing the services
```
kubectl get svc
```

More information on setting up Strongswan can be found on [IKS docs](https://console.bluemix.net/docs/containers/cs_vpn.html#vpn_configure)