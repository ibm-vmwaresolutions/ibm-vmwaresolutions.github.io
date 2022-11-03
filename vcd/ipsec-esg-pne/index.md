## vCD - IPSec Tunnel over IBM Private Network Endpoint (PNE) using ESG

Updated: 2020-10-21

In order to use a PNE for your IPSec tunnel between your IBM account to your IBM VMWare Solutions Shared virtual datacenter (vDC), you must first have a PNE ordered in your vDC.  See how to [Order IBM Private Network Endpoint (PNE)](https://ibm-vmwaresolutions.github.io/vcd/order-pne/).  Only one side requires a PNE.  

_If you are connecting your IBM account to your vDC, you only need the PNE deployed in one of your vDCs (you can share the network linked in your tunnel across multiple vDCs)._

This example will demonstrate how to connect two vCloud Director vDCs located in two different physical datacenters to each other using an IPSec tunnel and one PNE.  This allows bi-directional communication from virtual machines in both virtual datacenters using the IBM Cloud backbone.

The diagram below describes the flow of the connection between the two datacenters

<img src="images/1-vdc-pne-ipsec.png" width="1000" style="border: 1px solid black">

### The Datacenter information on the left:
- Physical datacenter: Dallas, TX, USA
- vCloud Director virtual datacenter: mwiles-dal-v10
- Edge Service Gateway address: 52.117.132.152
- VM network: 172.16.0.0/16
- VM (centos-dal): 172.16.0.2
- PNE address: 166.9.48.94

### The Datacenter information on the right:
- Physical datacenter: Frankfurt, Germany
- vCloud Director virtual datacenter: mwiles-fra-v10
- Edge Service Gateway address: 52.117.132.72
- VM network: 172.15.0.0/16
- VM (centos-fra): 172.15.0.2

## Configuring the Dallas vDC.

### Create a network.

<img src="images/2-dal-network.png" width="1000" style="border: 1px solid black">

Create a 172.16.0.0 Network
- Name: 172.16.0.0/16
- Gateway CIDR: 172.16.0.1/16
- Network Type: Routed
- Interface Type: Subinterface

<img src="images/3-dal-network.png" width="1000" style="border: 1px solid black">

Deploy at least 1 VM to test your tunnel.  172.16.0.2 will be our Dallas-side example.  Attach it to your network and ensure the network interface is configured properly.

<img src="images/4-dal-vm.png" width="1000" style="border: 1px solid black">

### Configure the Dallas Edge Service Gateway (ESG) services.

<img src="images/5-dal-esg.png" width="1000" style="border: 1px solid black">

### Firewall rules:
- CSE Inbound: This rule will allow all traffic from your PNE with destination of the ESG Service Address.  This information can be found on your IBM Cloud vDC information page.
  - Source: 166.9.48.94
  - Destination: 52.117.132.152
- Incoming IPSec: This rule allows the network from the Frankfurt side (yet to be created) access to this ESG network.
  - Source: 172.15.0.0/16
  - Destination: Any (this can be more restrictive if required)
- Outbound IPSec: This rule allows the Dallas ESG network outbound access.
  - Source: 172.16.0.0/16
  - Destination: Any (this can be more restrictive if required)

<img src="images/6-dal-esg-fw.png" width="1000" style="border: 1px solid black">

### Network Address Translation SourceNAT rule:
- _This rule is NOT required for the IPSec tunnel, but will be required if you want to access services on the IBM Cloud network (such as DNS, WSUS, Capsule services)._
  - Original: 172.16.0.0/16
  - Translated: 52.117.132.152

<img src="images/7-dal-esg-nat.png" width="1000" style="border: 1px solid black">

### IPSec VPN:
To enabled the service, toggle the status switch to show green.

<img src="images/8-dal-esg-ipsec.png" width="1000" style="border: 1px solid black">

### Create the IPSec VPN Site:
- Enabled: True
- Enable perfect forward secrecy (PFS): False
- Name: ipsec-dal-fra
- Local Id: ipsec-dal
- Local Endpoint: 52.117.132.152
- Local Subnets: 172.16.0.0/16
- Peer Id: ipsec-fra
- Peer Endpoint: 166.9.48.94
- Peer Subnets: 172.15.0.0/16
- Extensions: EMPTY
- Encryption Algorithm: AES
- Authentication: PKS
- Pre-Shared Key: _keycode used on both sides_
- Diffie-Hellman Group: DH5
- Digest Algorithm: SHA1
- IKE Option: IKEv1
- IKE Responder Only: Enabled (**this is on the PNE Side ONLY**)
- Session Type: Policy Based Session

_In our example we did not change the Encryption Algorithm, Authentication, Diffie-Hellman Group, Digest Algorithm, IKE Option, Session Type_

<img src="images/9-dal-esg-ipsec.png" width="1000" style="border: 1px solid black">

NOTE: Some additional items will be created from this IPSec VPN Site:
- Routes:

<img src="images/10-dal-esg-routes.png" width="1000" style="border: 1px solid black">
- Firewall Rules:

<img src="images/11-dal-esg-fw.png" width="1000" style="border: 1px solid black">

## Configuring the Frankfurt vDC.

### Create a network.

Create a 172.15.0.0 Network
- Name: 172.15.0.0/16
- Gateway CIDR: 172.15.0.1/16
- Network Type: Routed
- Interface Type: Subinterface

Deploy at least 1 VM to test your tunnel.  172.15.0.2 will be our Frankfurt-side example.  Attach it to your network and ensure the network interface is configured properly.

### Configure the Frankfurt Edge Service Gateway (ESG) services.

### Firewall rules:
- Incoming IPSec: This rule allows the network from the Dallas side access to this ESG network.
  - Source: 172.16.0.0/16
  - Destination: Any (this can be more restrictive if required)
- Outbound IPSec: This rule allows the Frankfurt ESG network outbound access.
  - Source: 172.15.0.0/16
  - Destination: Any (this can be more restrictive if required)

### Network Address Translation SourceNAT rule:
- _This rule is NOT required for the IPSec tunnel, but will be required if you want to access services on the IBM Cloud network (such as DNS, WSUS, Capsule services)._
  - Original: 172.15.0.0/16
  - Translated: 52.117.132.72

### IPSec VPN:
To enabled the service, toggle the status switch to show green.

### Create the IPSec VPN Site:
- Enabled: True
- Enable perfect forward secrecy (PFS): False
- Name: ipsec-fra-dal
- Name: ipsec-fra
- Local Id: ipsec-fra
- Local Endpoint: 52.117.132.72
- Local Subnets: 172.15.0.0/16
- Peer Id: ipsec-dal
- Peer Endpoint: 166.9.48.94
- Peer Subnets: 172.16.0.0/16
- Extensions: EMPTY
- Encryption Algorithm: AES
- Authentication: PKS
- Pre-Shared Key: _keycode used on both sides_
- Diffie-Hellman Group: DH5
- Digest Algorithm: SHA1
- IKE Option: IKEv1
- IKE Responder Only: **Leave Disabled**
- Session Type: Policy Based Session

## Test the tunnel

From the web-console, we log into the VMs in each side.  Then ensure we can ssh into the other VM as shown in the screenshot.

<img src="images/20-ssh-across-tunnel.png" width="1000" style="border: 1px solid black">

## Full config screenshots

Complete config from the FRA side of the tunnel

<img src="images/12-fra-esg-ipsec.png" width="1000" style="border: 1px solid black">

Complete config from the DAL side of the tunnel

<img src="images/13-dal-esg-ipsec.png" width="1000" style="border: 1px solid black">

_Note the information described in this example are guidelines.  There are multiple ways to configure the various parts of the example.  Please adjust accordingly for your needs._

[VMWare vCloud Director](https://ibm-vmwaresolutions.github.io/vcd/)<br/>
[Main Page](https://ibm-vmwaresolutions.github.io)