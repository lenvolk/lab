# Azure Networking Lab- IPSEC VPN (IKEv2) between Cisco CSR and Azure VPN Gateway- with BGP

This lab guide illustrates how to build a basic IPSEC VPN tunnel w/IKEv2 between a Cisco CSR and the Azure VPN gateway with BGP. This is for lab testing purposes only. All Azure configs are done in Azure CLI so you can change them as needed to match your environment. The entire lab including simulated on prem is done in Azure. No hardware required. Current CSR image used is 17.3.4a. All username/password are azureuser/Msft123Msft123. VM access except for CSR is done through serial console.

**Before deploying CSR in the next step, you may have to accept license agreement unless you have used it before. You can accomplish this through deploying a CSR in the portal or Powershell commands via Cloudshell**
<pre lang="...">
Sample Azure CLI:
az vm image terms accept --urn cisco:cisco-csr-1000v:17_3_4a-byol:latest
</pre>

# Base Topology
The lab deploys an Azure VPN gateway into a VNET. We will also deploy a Cisco CSR in a seperate VNET to simulate on prem.
![alt text](https://github.com/jwrightazure/lab/blob/master/images/csrvpnikev2.png)

**Build Resource Groups, VNETs and Subnets**
<pre lang="...">
RG="VPN-rg"
Location="eastus"

az group create --name VPN-rg --location $Location
az network vnet create --resource-group $RG --name Hub --location $Location --address-prefixes 10.0.0.0/16 --subnet-name HubVM --subnet-prefix 10.0.10.0/24
az network vnet subnet create --address-prefix 10.0.0.0/24 --name GatewaySubnet --resource-group $RG --vnet-name Hub
az network vnet create --resource-group $RG --name onprem --location $Location --address-prefixes 10.1.0.0/16 --subnet-name VM --subnet-prefix 10.1.10.0/24
az network vnet subnet create --address-prefix 10.1.0.0/24 --name zeronet --resource-group $RG --vnet-name onprem
az network vnet subnet create --address-prefix 10.1.1.0/24 --name onenet --resource-group $RG --vnet-name onprem
</pre>

**Build Azure and onprem Linux VMs**
<pre lang="...">
az network nic create --resource-group $RG -n HubVMNIC --location $Location --subnet HubVM --private-ip-address 10.0.10.10 --vnet-name Hub 
az vm create -n HubVM --resource-group $RG --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics HubVMNIC --no-wait --size Standard_D2as_v4

az network nic create --resource-group $RG -n onpremVMNIC --location $Location --subnet VM --private-ip-address 10.1.10.10 --vnet-name onprem 
az vm create -n onpremVM --resource-group $RG --image UbuntuLTS --admin-username azureuser --admin-password Msft123Msft123 --nics onpremVMNIC --no-wait --size Standard_D2as_v4
</pre>

**Build Public IPs for Azure VPN Gateway. The VPN GW will take 20+ minutes to deploy.**
<pre lang="...">
az network public-ip create --name Azure-VNGpubip --resource-group $RG --allocation-method Dynamic
az network vnet-gateway create --name Azure-VNG --public-ip-address Azure-VNGpubip --resource-group $RG --vnet Hub --gateway-type Vpn --vpn-type RouteBased --sku VpnGw3 --no-wait --asn 65001
</pre>

**Build onprem CSR. CSR image is specified from the Marketplace in this example.**
<pre lang="...">
az network public-ip create --name CSRPublicIP --resource-group $RG --idle-timeout 30 --allocation-method Static
az network nic create --name CSROutsideInterface --resource-group $RG --subnet zeronet --vnet onprem --public-ip-address CSRPublicIP --ip-forwarding true
az network nic create --name CSRInsideInterface --resource-group $RG --subnet onenet --vnet onprem --ip-forwarding true
az vm create --resource-group $RG --location $Location --name CSR --size Standard_D2as_v4 --nics CSROutsideInterface CSRInsideInterface  --image cisco:cisco-csr-1000v:17_3_4a-byol:latest --admin-username azureuser --admin-password Msft123Msft123 --no-wait
</pre>

**After the gateway and CSR have been created, document the public IP address for both. Value will be null until it has been successfully provisioned.**
<pre lang="...">
az network public-ip show --resource-group $RG -n Azure-VNGpubip --query "{address: ipAddress}"
az network public-ip show --resource-group $RG -n CSRPublicIP --query "{address: ipAddress}"
</pre>

**Document BGP peer IP and ASN**
<pre lang="...">
az network vnet-gateway list --query [].[name,bgpSettings.asn,bgpSettings.bgpPeeringAddress] -o table --resource-group $RG
</pre>

**Create a route table and routes for the Azure VNET with correct association. This is for the onprem simulation to route traffic to the CSR**
<pre lang="...">
az network route-table create --name vm-rt --resource-group $RG
az network route-table route create --name vm-rt --resource-group $RG --route-table-name vm-rt --address-prefix 10.0.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address 10.1.1.4
az network vnet subnet update --name VM --vnet-name onprem --resource-group $RG --route-table vm-rt
</pre>

**Create Local Network Gateway. The 192.168.1.1 addrees is the IP of the tunnel interface on the CSR in BGP ASN 65002.**
<pre lang="...">
az network local-gateway create --gateway-ip-address "insert CSR Public IP" --name to-onprem --resource-group $RG --local-address-prefixes 192.168.1.1/32 --asn 65001 --bgp-peering-address 192.168.1.1
</pre>

**Create VPN connections**
<pre lang="...">
az network vpn-connection create --name to-onprem --resource-group $RG --vnet-gateway1 Azure-VNG -l $Location --shared-key Msft123Msft123 --local-gateway2 to-onprem --enable-bgp
</pre>

**SSH to CSR public IP. Public IPs in the below config are an example.**
<pre lang="...">
!route for simulate on prem vm
ip route 10.1.10.0 255.255.255.0 10.1.1.1

crypto ikev2 proposal to-onprem-proposal
  encryption aes-cbc-256
  integrity  sha1
  group      2
  exit

crypto ikev2 policy to-onprem-policy
  proposal to-onprem-proposal
  match address local 10.1.0.4
  exit
  
crypto ikev2 keyring to-onprem-keyring
  peer "insert Azure VPN GW IP"
    address "insert Azure VPN GW IP"
    pre-shared-key Msft123Msft123
    exit
  exit

crypto ikev2 profile to-onprem-profile
  match address  local 10.1.0.4
  match identity remote address "insert Azure VPN GW IP" 255.255.255.255
  authentication remote pre-share
  authentication local  pre-share
  lifetime       3600
  dpd 10 5 on-demand
  keyring local  to-onprem-keyring
  exit

crypto ipsec transform-set to-onprem-TransformSet esp-gcm 256 
  mode tunnel
  exit

crypto ipsec profile to-onprem-IPsecProfile
  set transform-set  to-onprem-TransformSet
  set ikev2-profile  to-onprem-profile
  set security-association lifetime seconds 3600
  exit

int tunnel 1
  ip address 192.168.1.1 255.255.255.255
  tunnel mode ipsec ipv4
  ip tcp adjust-mss 1350
  tunnel source 10.1.0.4
  tunnel destination "insert Azure VPN GW IP"
  tunnel protection ipsec profile to-onprem-IPsecProfile
  exit

router bgp 65001
  bgp      log-neighbor-changes
  neighbor 10.0.0.254 remote-as 65515
  neighbor 10.0.0.254 ebgp-multihop 255
  neighbor 10.0.0.254 update-source tunnel 1

  address-family ipv4
    network 10.1.10.0 mask 255.255.255.0
    neighbor 10.0.0.254 activate    
    exit
  exit

!route BGP peer IP over the tunnel
ip route 10.0.0.254 255.255.255.255 Tunnel 1

</pre>

**Validate VPN connection status in Azure CLI**
<pre lang="...">
az network vpn-connection show --name to-onprem --resource-group $RG --query "{status: connectionStatus}"
</pre>


Key Cisco commands
- show interface tunnel 1
- show crypto session
- show crypto ipsec transform-set
- show crypto ikev2 proposal

**List BGP advertised routes per peer.**
<pre lang="...">
az network vnet-gateway list-advertised-routes -g $RG -n Azure-VNG --peer 192.168.1.1

PS Azure:\> az network vnet-gateway list-advertised-routes -g $RG -n Azure-VNG --peer 192.168.1.1
{
  "value": [
    {
      "asPath": "65001",
      "localAddress": "10.0.0.254",
      "network": "10.0.0.0/16",
      "nextHop": "10.0.0.254",
      "origin": "Igp",
      "sourcePeer": null,
      "weight": 0
    },
</pre>
**If you add a new prefix to your existing VNET, it will automatically be advertised to the CSR. Example: I added 172.16.1.0/24 to the VNET address space.**
<pre lang="...">
PS Azure:\> az network vnet-gateway list-advertised-routes -g $RG -n Azure-VNG --peer 192.168.1.1
{
  "value": [{
      "asPath": "65001",
      "localAddress": "10.0.0.254",
      "network": "172.16.1.0/24",
      "nextHop": "10.0.0.254",
      "origin": "Igp",
      "sourcePeer": null,
      "weight": 0
    }
 On CSR:
 CSR1#sh ip route 172.16.1.0
Routing entry for 172.16.1.0/24
  Known via "bgp 65002", distance 20, metric 0
  Tag 65001, type external
  Last update from 10.0.0.254 00:02:31 ago
  Routing Descriptor Blocks:
  * 10.0.0.254, from 10.0.0.254, 00:02:31 ago
      Route metric is 0, traffic share count is 1
      AS Hops 1
      Route tag 65001
      MPLS label: none
</pre>
**Advertise a new prefix from on prem over BGP**
<pre lang="...">
CSR:
CSR1(config)#int lo100
CSR1(config-if)#ip address 1.1.1.1 255.255.255.255
CSR1(config-if)#router bgp 65002
CSR1(config-router)#address-family ipv4
CSR1(config-router-af)#  network 1.1.1.1 mask 255.255.255.255

PS Azure:\> az network vnet-gateway list-learned-routes -g $RG -n Azure-VNG
{
      "asPath": "65002",
      "localAddress": "10.0.0.254",
      "network": "1.1.1.1/32",
      "nextHop": "192.168.1.1",
      "origin": "EBgp",
      "sourcePeer": "192.168.1.1",
      "weight": 32768
</pre>
