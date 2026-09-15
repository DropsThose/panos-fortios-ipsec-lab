# Multi-Vendor Site-to-Site IPsec Lab: PAN-OS ↔ FortiOS

## Overview

In this lab, I set up a site-to-site IPsec VPN tunnel between my Fortinet FortiGate 100E and a Palo Alto Firewall VM running on my Proxmox VE server. The purpose of this lab is to mimic a real-world situation in which an organization needs to set up a secure VPN connection over the insecure internet to facilitate the secure transfer of data or access to resources. I will walk through the steps I took to create a baseline configuration of the IPsec tunnel between the FortiGate 100E and the Palo Alto VM firewalls, as well as some troubleshooting I had to do along the way. 

<img width="800" height="281" alt="IPsec_lab_logical" src="https://github.com/user-attachments/assets/54a9663d-1964-40cc-b285-9e01e8c3b583" />

### Technology Utilized 
- Fortinet FortiGate 100E (7.2.13)
- Palo Alto Firewall VM (11.2.12)
- Cisco WS-C3750X-24T-S (15.2(4)E10)
- Proxmox VE (9.2.5)
- Ubuntu VMs (24.04.4 LTS)

## Baseline Setup
### 1. Network & Isolation
<img width="2061" height="343" alt="image" src="https://github.com/user-attachments/assets/5e5d4a48-4f1e-4135-b4a3-e89d63d67ea2" />

I created 2 VLANs on my Cisco switch. VLAN 100 (IPsec-lab-tunnel) for the point-to-point connection between the FortiGate 100E and PA VM, and VLAN 101 (IPsec-lab-LANB) for the 100E side Ubuntu client LAN B. 

<img width="745" height="330" alt="image" src="https://github.com/user-attachments/assets/c972e95f-05f0-499d-9d9d-03f96103fdb0" />

I added VLANs 100 and 101 to my trunk port that connects my Proxmox VE server to the Cisco switch (Gi1/0/1) to ensure that my Ubuntu VM traffic has a path to the switch. I also added VLAN 101 to my 100E trunk (Gi1/0/24) so that LAN B traffic could reach the 100E from the switch. I did not add VLAN 100 to my 100E trunk port, however, because I instead opted to run a separate, dedicated cable from a free port on my 100E to a free port on my Cisco switch to segment the IPsec tunnel from my other trunk port traffic. By segmenting my tunnel traffic to a dedicated port, I can implement tighter security controls without worrying about affecting other traffic on my trunk port.  

<img width="2311" height="918" alt="image" src="https://github.com/user-attachments/assets/c9ad57dd-42fa-4064-a867-a6f41f7cadd7" />

I chose to use port 9 on my 100E as my dedicated port for the IPsec tunnel. 

<img width="920" height="575" alt="image" src="https://github.com/user-attachments/assets/12650b94-5620-4468-b7f2-da0446e44b88" />

On the Cisco side, I chose to use Gi1/0/5 (IPsec-transport) as my dedicated port for the IPsec tunnel. I set up Gi1/0/5 as an access port to VLAN 100 to ensure that only traffic destined to the IPsec tunnel can flow through that port.

### 2. Physical/Virtual Link

<img width="2311" height="918" alt="651043247-c9ad57dd-42fa-4064-a867-a6f41f7cadd7" src="https://github.com/user-attachments/assets/dc32c9a7-df36-48f2-b1ab-98ab0113c8b6" />

On my 100E, I used 192.168.100.0/31 as the IP for my port 9 interface. This will be the IP that the 100E uses for the point-to-point connection to the PA VM for the tunnel. 

Normally, IPsec router peers would not be on the same subnet; instead, they would be on completely different networks reached via routing over the open internet. For the sake of this lab, I used a /31 to simulate the internet. 

Normally, a subnet needs to have a network ID and a broadcast address, but a /31 is unique in the sense that it has neither. RFC 3021 dictates that in a point-to-point connection, there really is no need for a network ID and broadcast address since it's just two devices connecting to each other. Before RFC 3021, people would have to use a /30, which would give them the same 2 usable IP addresses, but it would waste 2 addresses for the network ID and broadcast address. RFC 3021 allows the same point-to-point connection to occur while conserving 2 ipv4 addresses. Pretty cool! 

<img width="2558" height="1267" alt="image" src="https://github.com/user-attachments/assets/01b81b0e-8063-48b6-9c13-0e5a2313437c" />

On the PA VM side of the tunnel, I used ethernet1/1 as my tunnel interface and gave it 192.168.100.1/31. On the Proxmox VE backend, I configured the ethernet1/1 PA VM interface to be directly on VLAN 100. 

<img width="2559" height="1267" alt="image" src="https://github.com/user-attachments/assets/bdb8dfff-4c28-4395-8068-689b022b8581" />

With both point-to-point interfaces set up, I ran a ping test from the FortiGate to the PA VM to confirm connectivity before proceeding with IPsec configuration. 

<img width="2552" height="1261" alt="image" src="https://github.com/user-attachments/assets/65e45fbc-3ed7-42dd-b682-58679b7d402b" />

I ran a ping test from the PA VM to FortiGate. Success!

### 3. Phase 1 (IKE)

<img width="2555" height="1240" alt="image" src="https://github.com/user-attachments/assets/4bb16abd-cd29-43ac-a04e-1e5aee1bdeb5" />

On the PA VM, I configured the IKE Phase 1 proposal. I set AES-256-cbc for encryption, SHA-256 for authentication, and DH group 14. I also set the key lifetime to 8 hours. 

<img width="2546" height="1239" alt="image" src="https://github.com/user-attachments/assets/9fe059f9-8f5f-48b5-93ff-3dc770acb5ff" />

<img width="2552" height="1243" alt="image" src="https://github.com/user-attachments/assets/28b000b4-c91d-4f1b-9f28-30dcc03ff5aa" />

I configured the IKE Gateway, selecting ehternet1/1 as my interface, setting the local IP address to 192.168.100.1/31 and the peer address to 192.168.100.0. I chose to use a Pre-Shared Key for authentication. Under the Advanced Options, I set the IKE Crypto Profile to the previously created IKE Phase 1 proposal (to-fortigate-p1). 

<img width="1277" height="1272" alt="image" src="https://github.com/user-attachments/assets/cb4199b1-2554-4793-88d4-c3a23651c1a8" />

Jumping over to the 100E side, I matched the IKE phase 1 proposal settings I used on the PA VM. I also configured the same Pre-Shared Key and set the local and remote addresses.  

### 4. Phase 2 (IPsec)

<img width="2555" height="1239" alt="image" src="https://github.com/user-attachments/assets/c9714c55-c328-47fe-9359-de3a92e093ea" />

On the PA VM, I configured the IPsec phase 2 proposal with AES-256-cbc for encryption, SHA256 for authentication, and DH group 14. I set the IPsec Protocol to ESP and set the key lifetime to 1 hour.  

<img width="2553" height="1239" alt="image" src="https://github.com/user-attachments/assets/09c7f620-eed5-478a-88dd-d87adf1e1c1b" />

<img width="2551" height="1238" alt="image" src="https://github.com/user-attachments/assets/6a84bb79-d3cd-4635-9011-64b8844dcb5b" />

I created the IPsec tunnel on the PA VM, selecting the previously created phase 2 proposal (to-fortigate-p2), IKE gateway (to-fortigate), and the Ubuntu client subnets for both Ubuntu LANs (to-lanB). 

<img width="2557" height="1274" alt="image" src="https://github.com/user-attachments/assets/38dc9cd9-5157-484e-a91a-595b46c2cb90" />

On the 100E, I matched the PA VM phase 2 configuration.  

### 5. Routing & Policies 

<img width="2553" height="1266" alt="image" src="https://github.com/user-attachments/assets/cd0e4fae-8a78-4956-994a-507ade507e34" />

I created a VPN zone on the PA VM and added the tunnel.1 interface that was created after the previous phase 2 IPsec tunnel step.

<img width="2555" height="1267" alt="image" src="https://github.com/user-attachments/assets/ac2d17a7-2897-4c51-beeb-a7db7bfeb7c7" />

I created two firewall rules: one called "lana-to-vpn" that allows traffic from my Ubuntu LAN A to flow to the tunnel, and one called "vpn-to-lana" that allows traffic from the tunnel to flow to the Ubuntu LAN A.

<img width="2556" height="1266" alt="image" src="https://github.com/user-attachments/assets/cb43d784-9169-4e4c-bf08-6303420ca13f" />

I created a static route to ensure that traffic destined for Ubuntu LAN B would know to go through the tunnel interface. 

<img width="2557" height="1274" alt="image" src="https://github.com/user-attachments/assets/8cf2d6f9-a6a4-486c-8576-646ac4c3af04" />

On the 100E, I created two firewall rules: "lanb-to-vpn," which allows traffic from Ubuntu LAN B to flow to the tunnel, and "vpn-to-lanb," which allows traffic to flow from the tunnel to LAN B.  

<img width="2555" height="1270" alt="image" src="https://github.com/user-attachments/assets/8bea17b8-5f1b-415e-b351-072bdd7f769b" />

I created a static route on the 100E so that traffic destined for Ubuntu LAN A would know to go through the tunnel interface. 

### 6. Verification

<img width="2554" height="1258" alt="image" src="https://github.com/user-attachments/assets/0a65da7b-5ac1-48de-9d77-aa357e9ad31a" />

After configuring the previous steps, I saw that the IPsec tunnel was successfully established on the PA VM side. Notice, however, that the "PKT ENCAP" and "PKT DECAP" sit at zero within the screenshot. This is because although the tunnel is established, no traffic has been sent through it yet.  

<img width="2557" height="1271" alt="image" src="https://github.com/user-attachments/assets/81193852-8a40-42a0-b069-68dbf258479d" />

Here are the command outputs of "diagnose vpn tunnel list" and "diagnose vpn ike gateway list" on the 100E, showing that the IPsec tunnel is successfully established on the 100E side. I want to point out that the "dec:pkts/bytes" and "enc:pkts/bytes" values are zero in the screenshot. This mirrors the situation on the PA VM side: the tunnel is established, but no data has been sent over it yet.  

<img width="2554" height="1269" alt="image" src="https://github.com/user-attachments/assets/0e64bda1-9904-48fb-9342-7b7c089a489b" />

Here I have both Ubuntu VMs open. On the left, 172.16.2.2 is the 100E side LAN B. On the right, 172.16.1.2 is the PA VM side LAN A. From each Ubuntu VM, I pinged the other to show that traffic could successfully flow through the tunnel and connect to the VM on the other side.

<img width="2549" height="1264" alt="image" src="https://github.com/user-attachments/assets/c17b52e4-82ca-4d29-8caa-2bf4326981cd" />

Here is the same view of the PA VM tunnel info, but this time I want you to notice that the "PKT ENCAP" and "PKT DECAP" are no longer at zero! Our ping packets from the Ubuntu VMs successfully flowed through the tunnel and reached their intended target.

<img width="2554" height="1265" alt="image" src="https://github.com/user-attachments/assets/4dd4c5d9-cede-4f0b-adff-c7b501b9c2e1" />

Here are the command outputs of "diagnose vpn tunnel list" and "diagnose vpn ike gateway list" after the ping packets were sent. Notice "dec:pkts/bytes" and "enc:pkts/bytes" are now increased! This proves that the IPsec tunnel has been successfully established between the 100E and the PA VM, and that data can flow through it. 

## Troubleshooting 

- [ignoring IKEv2 request, no policy configured](troubleshooting/01-no-policy-configured.md)
