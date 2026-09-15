# Multi-Vendor Site-to-Site IPsec Lab: PAN-OS ↔ FortiOS

## Overview

In this lab, I set up a site-to-site IPsec VPN tunnel between my Fortinet FortiGate 100E and a Palo Alto Firewall VM running on my Proxmox VE server. The purpose of this lab is to mimic a real-world situation in which an organization needs to set up a secure VPN connection over the insecure internet to facilitate the secure transfer of data or access to resources. I will walk through the steps I took to create a baseline configuration of the IPsec tunnel between the FortiGate 100E and the Palo Alto VM firewalls, as well as some troubleshooting I had to do along the way. 

<img width="800" height="281" alt="IPsec_lab_logical" src="/assets/IPsec_tunnel_logical.jpg" />

### Technology Utilized 
- Fortinet FortiGate 100E (7.2.13)
- Palo Alto Firewall VM (11.2.12)
- Cisco WS-C3750X-24T-S (15.2(4)E10)
- Proxmox VE (9.2.5)
- Ubuntu VMs (24.04.4 LTS)

## Baseline Setup
### 1. Network & Isolation
<img width="2061" height="343" alt="image" src="/assets/cisco_vlan_brief.png" />

I created 2 VLANs on my Cisco switch. VLAN 100 (IPsec-lab-tunnel) for the point-to-point connection between the FortiGate 100E and PA VM, and VLAN 101 (IPsec-lab-LANB) for the 100E side Ubuntu client LAN B. 

<img width="745" height="330" alt="image" src="/assets/cisco_trunks.png" />

I added VLANs 100 and 101 to my trunk port that connects my Proxmox VE server to the Cisco switch (Gi1/0/1) to ensure that my Ubuntu VM traffic has a path to the switch. I also added VLAN 101 to my 100E trunk (Gi1/0/24) so that LAN B traffic could reach the 100E from the switch. I did not add VLAN 100 to my 100E trunk port, however, because I instead opted to run a separate, dedicated cable from a free port on my 100E to a free port on my Cisco switch to segment the IPsec tunnel from my other trunk port traffic. By segmenting my tunnel traffic to a dedicated port, I can implement tighter security controls without worrying about affecting other traffic on my trunk port.  

<img width="2311" height="918" alt="image" src="/assets/fortigate_interfaces.png" />

I chose to use port 9 on my 100E as my dedicated port for the IPsec tunnel. 

<img width="920" height="575" alt="image" src="/assets/cisco_interfaces.png" />

On the Cisco side, I chose to use Gi1/0/5 (IPsec-transport) as my dedicated port for the IPsec tunnel. I set up Gi1/0/5 as an access port to VLAN 100 to ensure that only traffic destined to the IPsec tunnel can flow through that port.

### 2. Physical/Virtual Link

<img width="2311" height="918" alt="651043247-c9ad57dd-42fa-4064-a867-a6f41f7cadd7" src="/assets/fortigate_interface_ip.png" />

On my 100E, I used 192.168.100.0/31 as the IP for my port 9 interface. This will be the IP that the 100E uses for the point-to-point connection to the PA VM for the tunnel. 

Normally, IPsec router peers would not be on the same subnet; instead, they would be on completely different networks reached via routing over the open internet. For the sake of this lab, I used a /31 to simulate the internet. 

Normally, a subnet needs to have a network ID and a broadcast address, but a /31 is unique in the sense that it has neither. RFC 3021 dictates that in a point-to-point connection, there really is no need for a network ID and broadcast address since it's just two devices connecting to each other. Before RFC 3021, people would have to use a /30, which would give them the same 2 usable IP addresses, but it would waste 2 addresses for the network ID and broadcast address. RFC 3021 allows the same point-to-point connection to occur while conserving 2 ipv4 addresses. Pretty cool! 

<img width="2558" height="1267" alt="image" src="/assets/PA_interfaces.png" />

On the PA VM side of the tunnel, I used ethernet1/1 as my tunnel interface and gave it 192.168.100.1/31. On the Proxmox VE backend, I configured the ethernet1/1 PA VM interface to be directly on VLAN 100. 

<img width="2559" height="1267" alt="image" src="/assets/PA_ping.png" />

With both point-to-point interfaces set up, I ran a ping test from the FortiGate to the PA VM to confirm connectivity before proceeding with IPsec configuration. 

<img width="2552" height="1261" alt="image" src="/assets/fortigate_ping.png" />

I ran a ping test from the PA VM to FortiGate. Success!

### 3. Phase 1 (IKE)

<img width="2555" height="1240" alt="image" src="/assets/PA_p1.png" />

On the PA VM, I configured the IKE Phase 1 proposal. I set AES-256-cbc for encryption, SHA-256 for authentication, and DH group 14. I also set the key lifetime to 8 hours. 

<img width="2546" height="1239" alt="image" src="/assets/PA_ike_gateway.png" />

<img width="2552" height="1243" alt="image" src="/assets/PA_ike_gateway2.png" />

I configured the IKE Gateway, selecting ehternet1/1 as my interface, setting the local IP address to 192.168.100.1/31 and the peer address to 192.168.100.0. I chose to use a Pre-Shared Key for authentication. Under the Advanced Options, I set the IKE Crypto Profile to the previously created IKE Phase 1 proposal (to-fortigate-p1). 

<img width="1277" height="1272" alt="image" src="/assets/fortigate_p1.png" />

Jumping over to the 100E side, I matched the IKE phase 1 proposal settings I used on the PA VM. I also configured the same Pre-Shared Key and set the local and remote addresses.  

### 4. Phase 2 (IPsec)

<img width="2555" height="1239" alt="image" src="/assets/PA_p2.png" />

On the PA VM, I configured the IPsec phase 2 proposal with AES-256-cbc for encryption, SHA256 for authentication, and DH group 14. I set the IPsec Protocol to ESP and set the key lifetime to 1 hour.  

<img width="2553" height="1239" alt="image" src="/assets/PA_tunnel.png" />

<img width="2551" height="1238" alt="image" src="/assets/PA_tunnel2.png" />

I created the IPsec tunnel on the PA VM, selecting the previously created phase 2 proposal (to-fortigate-p2), IKE gateway (to-fortigate), and the Ubuntu client subnets for both Ubuntu LANs (to-lanB). 

<img width="2557" height="1274" alt="image" src="/assets/fortigate_p2.png" />

On the 100E, I matched the PA VM phase 2 configuration.  

### 5. Routing & Policies 

<img width="2553" height="1266" alt="image" src="/assets/PA_zones.png" />

I created a VPN zone on the PA VM and added the tunnel.1 interface that was created after the previous phase 2 IPsec tunnel step.

<img width="2555" height="1267" alt="image" src="/assets/PA_policies.png" />

I created two firewall rules: one called "lana-to-vpn" that allows traffic from my Ubuntu LAN A to flow to the tunnel, and one called "vpn-to-lana" that allows traffic from the tunnel to flow to the Ubuntu LAN A.

<img width="2556" height="1266" alt="image" src="/assets/PA_routes.png" />

I created a static route to ensure that traffic destined for Ubuntu LAN B would know to go through the tunnel interface. 

<img width="2557" height="1274" alt="image" src="/assets/fortigate_policies.png" />

On the 100E, I created two firewall rules: "lanb-to-vpn," which allows traffic from Ubuntu LAN B to flow to the tunnel, and "vpn-to-lanb," which allows traffic to flow from the tunnel to LAN B.  

<img width="2555" height="1270" alt="image" src="/assets/fortigate_routes.png" />

I created a static route on the 100E so that traffic destined for Ubuntu LAN A would know to go through the tunnel interface. 

### 6. Verification

<img width="2554" height="1258" alt="image" src="/assets/PA_tunnel_est.png" />

After configuring the previous steps, I saw that the IPsec tunnel was successfully established on the PA VM side. Notice, however, that the "PKT ENCAP" and "PKT DECAP" sit at zero within the screenshot. This is because although the tunnel is established, no traffic has been sent through it yet.  

<img width="2557" height="1271" alt="image" src="/assets/fortigate_tunnel_est.png" />

Here are the command outputs of "diagnose vpn tunnel list" and "diagnose vpn ike gateway list" on the 100E, showing that the IPsec tunnel is successfully established on the 100E side. I want to point out that the "dec:pkts/bytes" and "enc:pkts/bytes" values are zero in the screenshot. This mirrors the situation on the PA VM side: the tunnel is established, but no data has been sent over it yet.  

<img width="2554" height="1269" alt="image" src="/assets/ubuntu_pings.png" />

Here I have both Ubuntu VMs open. On the left, 172.16.2.2 is the 100E side LAN B. On the right, 172.16.1.2 is the PA VM side LAN A. From each Ubuntu VM, I pinged the other to show that traffic could successfully flow through the tunnel and connect to the VM on the other side.

<img width="2549" height="1264" alt="image" src="/assets/PA_tunnel_after.png" />

Here is the same view of the PA VM tunnel info, but this time I want you to notice that the "PKT ENCAP" and "PKT DECAP" are no longer at zero! Our ping packets from the Ubuntu VMs successfully flowed through the tunnel and reached their intended target.

<img width="2554" height="1265" alt="image" src="/assets/fortigate_tunnel_after.png" />

Here are the command outputs of "diagnose vpn tunnel list" and "diagnose vpn ike gateway list" after the ping packets were sent. Notice "dec:pkts/bytes" and "enc:pkts/bytes" are now increased! This proves that the IPsec tunnel has been successfully established between the 100E and the PA VM, and that data can flow through it. 

## Troubleshooting 

- [ignoring IKEv2 request, no policy configured](troubleshooting/01-no-policy-configured.md)
