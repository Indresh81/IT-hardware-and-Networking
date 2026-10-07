# Network & Infrastructure Engineering Portfolio

Hands-on networking and infrastructure labs: server virtualisation, Cisco switching and routing, EtherChannel stacking, FortiGate firewalls and HA, Meraki wireless, Cisco ISE access control and packet analysis.

**Indresh S** | B.E. Electronics and Communication Engineering (2022-2026)
Email: indresh8105@gmail.com | [LinkedIn](https://www.linkedin.com/in/indresh-senthilkumar) | [GitHub](https://github.com/Indresh81)

---

## What's in here

- VMware ESXi 8.0 on a physical HPE ProLiant DL360 Gen9, including a real DIMM fault found at POST
- Windows Server AD/DHCP/DNS failover with RADIUS (NPS) authentication for switch SSH
- Cisco switching and routing in Packet Tracer (STP failover)
- Cisco 1921 router and Catalyst 3560-CX on real hardware, with an inter-VLAN ACL policy
- Catalyst 3850 stack with dual LACP EtherChannels
- FortiGate 300D Active-Passive HA cluster
- FortiGate hairpin (U-turn) NAT with a Windows IIS server
- FortiGate 40F + Meraki MS130-8P VLAN trunking
- Meraki MR36 wireless access points
- Cisco ISE: EAP-TLS network access with dynamic VLAN assignment
- Wireshark captures: ARP, ICMP and switch MAC learning
- Layer 2 loop test: STP enabled vs disabled

---

## Lab environment note

Some labs ran in Packet Tracer and the rest on physical devices. Each lab below says which one it used.

---

## Projects

| #  | Project | Description | Status |
|----|---------|-------------|--------|
| 1  | [ESXi on HPE ProLiant](Windows_Server_ESXi_Lab) | ESXi 8.0 U3 on DL360 Gen9, iLO 4, VMFS6 datastore, POST diagnostics | ✅ Complete |
| 2  | [AD Failover & AAA Lab](AD_Failover_and_AAA_Lab) | Multi-DC Windows Server, RADIUS failover for switch SSH | ✅ Complete |
| 3  | [Cisco Labs (Packet Tracer)](Cisco_Labs) | Spanning Tree root election and BLK to FWD failover | ✅ Complete |
| 4  | [Cisco Real Hardware](Cisco_Real_Hardware) | 1921 ISR + Catalyst 3560-CX, VLANs, inter-VLAN ACL isolation | ✅ Complete |
| 5  | [3850 Stack EtherChannel Lab](Cisco_3850_EtherChannel_Stack_Lab) | 3850 stack, LACP Port-channels, VLAN 10/20 SVIs, SPAN monitoring | ✅ Complete |
| 6  | [STP Loop Test Lab](STP_Loop_Test_Lab) | Two 3560s with a dual-link loop, STP enabled vs disabled | ✅ Complete |
| 7  | [FortiGate Active-Passive HA](FortiGate_Active-Passive-HA) | 2x FortiGate 300D, HA sync, primary/secondary failover | ✅ Complete |
| 8  | [FortiGate Hairpin NAT](FortiGate_Hairpin_NAT) | Internal client reaches IIS through its external VIP | ✅ Complete |
| 9  | [FortiGate + Meraki VLAN Trunking](FortiGate_Meraki_VLAN_Trunking) | FortiGate 40F + MS130-8P, VLAN 10 IT, VLAN 20 HR, native VLAN 999 | ✅ Complete |
| 10 | [Meraki MR36 Wireless](Cisco_Meraki_MR36_Wireless) | Two MR36 access points on a Meraki switch | ✅ Complete |
| 11 | [Cisco ISE Access Control](Cisco_ISE_Access_Control) | EAP-TLS with approved-group checks and dynamic VLANs | ✅ Complete |
| 12 | [Wireshark Traffic Analysis](Wireshark_Traffic_Analysis) | ARP resolution, ICMP, switch MAC address learning | ✅ Complete |

---

## Lab highlights

### ESXi on HPE ProLiant DL360 Gen9

ESXi 8.0.3 installed bare-metal on a DL360 Gen9 with two Xeon E5-2680 v4 CPUs and 31.9 GiB RAM, managed through iLO 4 and the ESXi Host Client. POST showed an uncorrectable memory error on Processor 2, DIMM 12, which was documented as a real hardware fault. The host has a 2.06 TB VMFS6 datastore.

![Server front panel](Windows_Server_ESXi_Lab/Screenshots/server-front.jpg)
![Server internals](Windows_Server_ESXi_Lab/Screenshots/server-internals.jpg)
![ESXi boot screen](Windows_Server_ESXi_Lab/Screenshots/esxi-boot.jpg)
![POST screen with iLO IP and DIMM error](Windows_Server_ESXi_Lab/Screenshots/post-screen-ilo-dimm-error.jpg)
![ESXi Host Client dashboard](Windows_Server_ESXi_Lab/Screenshots/esxi-dashboard.png)
![VMFS6 datastore](Windows_Server_ESXi_Lab/Screenshots/storage.png)

---

### AD Failover & RADIUS AAA

Switch SSH logins are authenticated by RADIUS. With the first RADIUS server unreachable, the switch retries, then fails over to the second server and receives an Access-Accept with privilege level 15.

![RADIUS failover debug with Access-Accept](AD_Failover_and_AAA_Lab/Screenshots/Lab-02_013_SSH-Debug-Failover-to-DC02-Access-Accept.png)

---

### Cisco Spanning Tree Failover (Packet Tracer)

After the root port Fa0/1 goes down, the blocked alternate port Fa0/2 becomes the new root port and moves to forwarding.

![STP failover, BLK to FWD](Cisco_Labs/Screenshots/sw1-failover-blk-to-fwd.png)

---

### Cisco Real Hardware

A Cisco 1921 ISR and a Catalyst 3560-CX, connected by console cable and used as a router plus Layer 3 switch pair. An ACL policy isolates the IT, HR and other VLANs from each other, verified with sourced pings from each SVI.

![Router and switch on the bench](Cisco_Real_Hardware/Screenshots/hardware-full-lab-overview.jpg)
![ACL isolation verified with sourced pings](Cisco_Real_Hardware/Screenshots/switch-acl-isolation-verified.png)

---

### Catalyst 3850 Stack with LACP EtherChannels

The stack shows an Active and a Standby member, both stack ports OK, and two LACP port-channels bundled across members (Gi1/0/x and Gi2/0/x).

![Stack and EtherChannel baseline](Cisco_3850_EtherChannel_Stack_Lab/Screenshots/08-baseline-hostname-switch-etherchannel.png)

---

### FortiGate Active-Passive HA

Two FortiGate 300D units in an HA cluster. The first view shows both synchronized with FW1-HA as Primary (priority 200). After a failover, FW2-HA is Primary and FW1-HA is listed as Secondary, Out of sync while it rejoins.

![HA cluster synchronized](FortiGate_Active-Passive-HA/Screenshots/12_ha_cluster_both_synchronized.png)
![FW1 out of sync after power restore](FortiGate_Active-Passive-HA/Screenshots/35_fw1_out_of_sync_after_power_restore.png)

---

### FortiGate Hairpin NAT with Windows IIS

An internal client reaches the IIS server (192.168.18.100) through its external VIP (172.18.2.11). The server-side capture shows the request arriving from the FortiGate LAN address 192.168.18.1, which confirms the source was translated as well as the destination.

![Internal client reaching IIS through the VIP](FortiGate_Hairpin_NAT/Screenshots/22-After-Hairpin-VIP-Access-Success.png)
![IIS-side packet capture](FortiGate_Hairpin_NAT/Screenshots/29-After-Hairpin-IIS-Server-Capture.png)

---

### Meraki MR36 Wireless

Two MR36 access points powered from a Meraki switch.

![Two MR36 access points](Cisco_Meraki_MR36_Wireless/Screenshots/01-Two-Cisco-Meraki-MR36-Access-Points.jpeg)

---

### Cisco ISE: EAP-TLS Access Control

ISE live logs show the IT user authorised into VLAN 10 and the HR user authorised into VLAN 20. An earlier HR attempt was denied until the endpoint met the approved-group condition.

![ISE live logs, IT and HR results](Cisco_ISE_Access_Control/Screenshots/44-Cisco-ISE-Live-Logs-IT-and-HR-EAPTLS-Results.png)

---

### Wireshark Traffic Analysis

The ARP broadcast resolving a MAC address before the first ping, and the switch MAC address table filling with dynamic entries once traffic flows.

![ARP request before ICMP](Wireshark_Traffic_Analysis/Screenshots/02_arp_request_frame16.png)
![MAC table after ping](Wireshark_Traffic_Analysis/Screenshots/19_mac_table_after_ping.png)

---

## Skills

| Category | Tools & Technologies |
|---|---|
| Virtualisation & servers | VMware ESXi 8.0, HPE ProLiant DL360 Gen9, iLO 4 |
| Windows Server | AD DS, DNS, DHCP, NPS/RADIUS |
| Cisco switching | VLANs, SVI, STP, LACP EtherChannel, StackWise, SPAN |
| Cisco routing & security | NAT, extended ACLs, SSH, Cisco 1921 ISR, Catalyst 3560-CX/3850 |
| Firewalls | FortiGate 300D/40F, HA clustering, VIP and hairpin NAT |
| Cloud & wireless | Meraki MS130-8P, MR36, 802.1Q trunking |
| Access control | Cisco ISE, EAP-TLS, dynamic VLAN assignment, RADIUS |
| Analysis & tools | Wireshark, PuTTY, Packet Tracer |

---

## Key lessons learned

- Check Layer 1 and POST output first: the iLO IP and a failed DIMM both showed up before the OS loaded.
- An ACL's implicit deny can silently block traffic, so test every VLAN pair in both directions.
- A trunk being up does not mean every VLAN is passing.
- A passing ping from one direction does not prove a policy works; test from each side.
- A firewall's HA "synchronized" status and a unit rejoining "out of sync" are different states and need separate checks.
- A successful web page proves less than the translated addresses seen in a server-side capture.
- A valid certificate proves identity, but group membership can still deny access.

---

*Built from hands-on training and lab work. Updated as new labs are completed.*
