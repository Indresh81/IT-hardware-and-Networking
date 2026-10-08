# FortiGate Hairpin NAT (U-Turn NAT) Lab

## Overview

This lab demonstrates **Hairpin NAT**, also called **U-Turn NAT**, on a FortiGate firewall using an internal IIS web server.

The lab was built to prove three different traffic paths:

1. **Direct LAN access** — an internal client accesses the IIS server using its private IP address.
2. **External VIP access** — an external client accesses the IIS server through a FortiGate Virtual IP (VIP).
3. **Internal Hairpin NAT access** — an internal client accesses the same IIS server by using the external/VIP address instead of the server's private address.

The key learning point is that a working external VIP does **not automatically mean** an internal LAN client can use that same VIP. Hairpin NAT requires FortiGate to correctly recognize the VIP, match the required firewall policy, translate the destination, and forward the session back toward the LAN.

This repository contains the full screenshot evidence for server preparation, FortiGate interface configuration, VIP creation, firewall policies, failure before Hairpin NAT, successful access after the Hairpin policies were added, and packet captures showing the traffic behavior.

---

# Main Learning Objectives

- Understand the difference between normal NAT, destination NAT, VIP/port forwarding, and Hairpin NAT.
- Configure a Windows IIS server as the published internal web server.
- Verify direct HTTP access before adding NAT.
- Configure the FortiGate WAN and LAN interfaces.
- Build a FortiGate software switch for LAN connectivity.
- Create firewall address objects.
- Create an HTTP Virtual IP.
- Publish the IIS server to an external network.
- Verify external access to the VIP.
- Demonstrate that internal access to the VIP initially fails.
- Add the required Hairpin NAT firewall policies.
- Verify internal access to the same VIP after configuration.
- Use FortiGate policy hit counters as proof of policy matching.
- Use Wireshark captures to compare direct LAN, external VIP, failed Hairpin, and successful Hairpin sessions.

---

# Lab Topology

```mermaid
flowchart LR
    EXT[External PC<br/>172.18.2.5/28]
    WAN[FortiGate WAN<br/>port1: 172.18.2.2/28]
    VIP[VIP_IIS_HTTP<br/>172.18.2.11:80<br/>→ 192.168.18.100:80]

    subgraph FG["FortiGate 300D"]
        WAN
        VIP
        LAN[LAN_INTERFACE<br/>Software Switch<br/>port3 + port4<br/>192.168.18.1/24]
    end

    PC2[Internal PC2<br/>192.168.18.10/24]
    IIS[IIS Web Server<br/>192.168.18.100/24]

    EXT -->|HTTP to 172.18.2.11| WAN
    WAN --> VIP
    VIP --> IIS
    PC2 --> LAN
    LAN --> IIS
    PC2 -. Hairpin HTTP to 172.18.2.11 .-> VIP
```

---

# IP Addressing Used

| Device / Interface | Address | Purpose |
|---|---|---|
| FortiGate management | `192.168.1.99/24` | GUI administration shown in screenshots |
| FortiGate WAN `port1` | `172.18.2.2/28` | External/WAN-facing interface |
| External PC | `172.18.2.5/28` | Tests external VIP access |
| VIP external address | `172.18.2.11` | Public/test-side address used to reach IIS |
| FortiGate LAN software switch | `192.168.18.1/24` | Default gateway for internal LAN |
| PC2 | `192.168.18.10/24` | Internal Hairpin NAT test client |
| IIS Server | `192.168.18.100/24` | Internal web server |
| HTTP service | TCP `80` | Published IIS service |

---

# Traffic Paths Tested

## Direct LAN path

```text
PC2 192.168.18.10
        ↓
192.168.18.100:80
        ↓
IIS Server
```

This proves that the internal client and server can communicate directly without NAT.

## External VIP path

```text
External PC 172.18.2.5
        ↓
VIP 172.18.2.11:80
        ↓ DNAT
192.168.18.100:80
        ↓
IIS Server
```

This proves the VIP and WAN-to-LAN policy work for external users.

## Hairpin NAT path

```text
PC2 192.168.18.10
        ↓
VIP 172.18.2.11:80
        ↓
FortiGate receives traffic from LAN
        ↓
VIP destination translation
        ↓
192.168.18.100:80
        ↓
Traffic returns toward LAN
        ↓
IIS Server
```

The packet effectively enters FortiGate from the LAN and is sent back toward the LAN after destination translation. This is why it is called **Hairpin**, **U-Turn**, or **NAT loopback** traffic.

---

# Part 1 — Prepare the IIS Web Server

## 1. Install IIS Windows Features

The first step was to prepare the Windows server that would later be published through FortiGate.

![IIS Windows Features](Screenshot/01-IIS-Windows-Features.png)

The Windows Features window shows the IIS components being enabled.

### Why IIS is required

Hairpin NAT itself is a network function, but a real application is needed to prove that the translated traffic reaches the final server successfully.

In this lab, HTTP on TCP port 80 is used as the test application.

---

## 2. Verify IIS Server IP Configuration

The IIS server network settings were checked with `ipconfig`.

![IIS Server IP Configuration](Screenshot/02-IIS-Server-IP-Configuration.png)

The screenshot shows:

```text
IPv4 Address    : 192.168.18.100
Subnet Mask     : 255.255.255.0
Default Gateway : 192.168.18.1
```

### Why the default gateway is important

The IIS server must send traffic for remote networks back to the FortiGate.

Its gateway therefore points to:

```text
192.168.18.1
```

which is the FortiGate LAN interface.

Without a correct gateway, requests may reach the IIS server but replies to non-local clients can fail.

---

## 3. Test the Default IIS Page

The IIS site was tested directly before introducing NAT.

![Default IIS page](Screenshot/03-IIS-Default-Page-Initial-Test.png)

This screenshot confirms that IIS was running and the HTTP service was accessible.

### Why this baseline matters

A NAT lab should not begin by troubleshooting NAT before proving that the application itself works.

The logical troubleshooting order is:

```text
Application works locally
        ↓
Application works over LAN
        ↓
VIP works from external side
        ↓
Hairpin NAT works internally
```

---

## 4. Verify the Custom IIS Test Page

A custom test page was loaded through `localhost`.

![Custom IIS local test](Screenshot/04-IIS-Custom-Page-Localhost-Test.png)

The page identifies the server as:

```text
192.168.18.100
```

This custom page provides clearer visual evidence later when the same server is accessed through different traffic paths.

---

# Part 2 — Verify FortiGate Interfaces

## 5. FortiGate Interface Summary

The FortiGate interface page was checked before NAT configuration.

![FortiGate interface summary](Screenshot/05-FortiGate-Interface-Summary.png)

Important visible settings include:

```text
WAN (port1)    : 172.18.2.2/28
LAN_INTERFACE  : 192.168.18.1/24
LAN members    : port3, port4
Management     : 192.168.1.99/24
```

### What this proves

The screenshot establishes the Layer-3 addressing used by the lab.

The firewall has connectivity to both:

```text
External network : 172.18.2.0/28
Internal LAN     : 192.168.18.0/24
```

This allows the FortiGate to sit between the external test client and the internal IIS server.

---

## 6. LAN Software Switch Configuration

The LAN side was configured as a FortiGate **Software Switch** named `LAN_INTERFACE`.

![LAN software switch](Screenshot/06-LAN-Software-Switch-Configuration.png)

Visible configuration:

```text
Alias             : LAN_INTERFACE
Type              : Software Switch
Members           : port3, port4
IP/Netmask        : 192.168.18.1/255.255.255.0
Role              : LAN
```

### Why a software switch is useful here

By placing `port3` and `port4` inside the same software switch, multiple physical FortiGate ports can participate in the same logical LAN.

Both internal devices can therefore exist in:

```text
192.168.18.0/24
```

while using the FortiGate as their gateway.

---

# Part 3 — Create Address Objects

## 7. VIP External Address Object

An address object was created for the VIP-side address.

![VIP external address object](Screenshot/07-VIP-External-Address-Object.png)

This object is later used in policy matching.

---

## 8. External PC Address Object

The external test client was also defined as a FortiGate address object.

![External PC address object](Screenshot/08-External-PC-Address-Object.png)

The external PC is used to prove that the normal WAN-to-LAN destination NAT is working before Hairpin NAT is attempted.

---

## 9. LAN Subnet Address Object

The internal LAN subnet was created as an address object.

![LAN subnet address object](Screenshot/09-LAN-Subnet-Address-Object.png)

The internal subnet is:

```text
192.168.18.0/24
```

This object is later used as the source for the Hairpin NAT firewall policies.

---

# Part 4 — Configure the IIS Virtual IP

## 10. Create `VIP_IIS_HTTP`

A FortiGate Virtual IP was configured to translate the external address to the private IIS server.

![IIS HTTP VIP configuration](Screenshot/10-IIS-HTTP-VIP-Configuration.png)

Visible VIP settings:

```text
Name                : VIP_IIS_HTTP
Type                : Static NAT
External IP          : 172.18.2.11
Mapped IP            : 192.168.18.100
Port Forwarding      : Enabled
Protocol             : TCP
External service port: 80
Map to port           : 80
```

### What the VIP does

The VIP performs destination translation:

```text
172.18.2.11:80
        ↓ DNAT
192.168.18.100:80
```

The client does not need to know the IIS server's private address.

It only connects to:

```text
http://172.18.2.11
```

FortiGate translates that connection to the real server.

---

# Part 5 — Publish IIS to the External Network

## 11. WAN-to-LAN IIS Policy

A firewall policy was created to allow the external PC to use the IIS VIP.

![WAN to LAN IIS policy](Screenshot/11-WAN-to-LAN-IIS-Policy.png)

Visible policy details include:

```text
Name               : WAN to LAN
Incoming Interface : WAN (port1)
Outgoing Interface : LAN_INTERFACE (lan)
Source             : EX PC
Destination        : VIP_IIS_HTTP
Schedule           : always
Service            : HTTP
Action             : ACCEPT
```

### Important concept

The VIP performs address translation, but the VIP alone does not automatically authorize traffic.

FortiGate still requires a matching firewall policy.

Therefore:

```text
VIP    = How the destination is translated
Policy = Whether the traffic is permitted
```

Both are required.

---

## 12. Verify the External PC IP

The external client configuration was checked.

![External PC IP configuration](Screenshot/12-External-PC-IP-Configuration.png)

The screenshot shows:

```text
IPv4 Address    : 172.18.2.5
Subnet Mask     : 255.255.255.240
Default Gateway : 172.18.2.1
```

Because the VIP is `172.18.2.11`, both addresses exist in the same `/28` external test network.

---

## 13. Verify External VIP Web Access

The external PC successfully opened:

```text
http://172.18.2.11
```

![External VIP access success](Screenshot/13-External-VIP-Access-Success.png)

The page confirms that the external/VIP address successfully reached the IIS server at:

```text
192.168.18.100
```

### What this proves

At this point, the following are verified:

- IIS is running.
- The FortiGate WAN is reachable.
- `VIP_IIS_HTTP` is correct.
- TCP port 80 mapping is correct.
- The WAN-to-LAN policy is matching.
- FortiGate can forward the translated request to IIS.
- The return path works.

---

## 13A. Verify TCP Port 80

A PowerShell TCP test was also performed against the VIP.

![External VIP TCP 80 success](Screenshot/13A-External-VIP-TCP-Port-80-Test-Success.png)

The test uses:

```powershell
Test-NetConnection 172.18.2.11 -Port 80
```

The screenshot shows:

```text
TcpTestSucceeded : True
```

### Why this test is useful

A browser test proves application access, while `Test-NetConnection` gives a direct TCP-level verification.

This helps distinguish:

```text
TCP connectivity problem
vs.
Web browser/application problem
```

---

# Part 6 — Verify Direct Internal LAN Access

## 14. PC2 IP Configuration

PC2 was configured as the internal Hairpin NAT client.

![PC2 IP configuration CLI](Screenshot/14-PC2-IP-Configuration-CLI.png)

The visible values include:

```text
IPv4 Address    : 192.168.18.10
Subnet Mask     : 255.255.255.0
Default Gateway : 192.168.18.1
```

---

## 15. PC2 GUI Network Details

The same addressing was verified through the Windows GUI.

![PC2 IP configuration GUI](Screenshot/15-PC2-IP-Configuration-GUI.png)

Using both CLI and GUI verification provides additional evidence that the client was correctly placed on the internal LAN.

---

## 16. Test Direct Connectivity to IIS

Before Hairpin NAT was tested, PC2 directly contacted the IIS server private address.

![Direct LAN connectivity and HTTP test](Screenshot/16-Direct-LAN-Connectivity-and-HTTP-Test.png)

The screenshot demonstrates both:

```text
ping 192.168.18.100
```

and a TCP port test to:

```text
192.168.18.100:80
```

### Why this step is critical

If PC2 cannot reach `192.168.18.100` directly, a later Hairpin NAT failure cannot be confidently blamed on Hairpin NAT.

This step proves basic LAN connectivity first.

---

## 17. Direct LAN Web Access

PC2 successfully accessed the web server through its private address.

![Direct LAN web access](Screenshot/17-Direct-LAN-Web-Access-Success.png)

The direct URL is:

```text
http://192.168.18.100
```

This confirms:

```text
PC2 → LAN → IIS
```

is working normally.

---

# Part 7 — Demonstrate the Hairpin NAT Problem

## 18. Internal Access to the VIP Fails Before Hairpin Configuration

PC2 then attempted to access the IIS server by using the external/VIP address:

```text
http://172.18.2.11
```

![Before Hairpin VIP access failed](Screenshot/18-Before-Hairpin-VIP-Access-Failed.png)

The browser shows that the site could not be reached.

### Why this failure is important

The failure provides the baseline proving that:

- Direct LAN access works.
- External VIP access works.
- But internal-to-VIP access does not yet work.

That isolates the missing function to **Hairpin NAT / U-Turn NAT handling**.

This is stronger evidence than simply showing the final successful result.

---

# Part 8 — Configure Hairpin NAT Policies

## 19. Hairpin Policy 1 — LAN to WAN

The first Hairpin-related policy was created.

![Hairpin policy 1](Screenshot/19-Hairpin-Policy-1-LAN-to-WAN.png)

Visible settings:

```text
Name               : HPN-1-LAN-to-VIP-Outside
Incoming Interface : LAN_INTERFACE (lan)
Outgoing Interface : WAN (port1)
Source             : LAN_192.168.18.0_24
Destination        : ADDR_WAN_VIP_HTTP
Schedule           : always
Service            : HTTP
Action             : ACCEPT
NAT                : Enabled
IP Pool            : Use Outgoing Interface Address
```

### Why this policy exists

The internal client is deliberately trying to access an address that belongs to the external/WAN side.

This policy permits the LAN-originated HTTP request toward that external VIP-side address.

The screenshot also shows NAT enabled, meaning source translation is part of this Hairpin path.

---

## 20. Hairpin Policy 2 — LAN to LAN VIP

A second policy was created for the translated traffic.

![Hairpin policy 2](Screenshot/20-Hairpin-Policy-2-LAN-to-LAN.png)

Visible settings:

```text
Name               : HPN-2-LAN-to-IIS-VIP
Incoming Interface : LAN_INTERFACE (lan)
Outgoing Interface : LAN_INTERFACE (lan)
Source             : LAN_192.168.18.0_24
Destination        : VIP_IIS_HTTP
Schedule           : always
Service            : HTTP
Action             : ACCEPT
NAT                : Disabled
```

### Why LAN-to-LAN is important

After FortiGate recognizes the VIP, the real server is still on the same LAN side as the client.

The traffic therefore makes a U-turn:

```text
LAN client
   ↓
FortiGate
   ↓
VIP translation
   ↓
same LAN interface
   ↓
IIS server
```

This is the practical meaning of **Hairpin NAT** in this lab.

---

## 21. Verify Policy Order and Hit Counters

The firewall policy list was checked after configuration.

![Policy summary and hit counters](Screenshot/21-FortiGate-Policy-Summary-and-Hit-Counters.png)

### Why hit counters matter

A configuration can look correct but still never match real traffic.

Policy hit counters provide evidence that FortiGate is actually selecting the expected rule.

When troubleshooting Hairpin NAT, check:

```text
Is the expected policy receiving hits?
```

If the hit count stays at zero, the problem may involve:

- Wrong incoming interface.
- Wrong outgoing interface.
- Wrong source object.
- Wrong destination object.
- Wrong service.
- Policy order.
- VIP mismatch.

---

# Part 9 — Verify Hairpin NAT Success

## 22. Internal Client Successfully Uses the External VIP

After the Hairpin policies were added, PC2 accessed:

```text
http://172.18.2.11
```

successfully.

![After Hairpin VIP access success](Screenshot/22-After-Hairpin-VIP-Access-Success.png)

The web page identifies:

```text
IIS server  : 192.168.18.100
Address used: 172.18.2.11
```

### What this proves

This screenshot provides the main proof of the lab:

```text
Internal PC2
192.168.18.10
     ↓
External/VIP address
172.18.2.11:80
     ↓
FortiGate Hairpin NAT
     ↓
Internal IIS server
192.168.18.100:80
```

**Hairpin NAT result: PASS**

---

# Part 10 — Packet Analysis

Packet captures were collected to compare the different traffic flows and provide protocol-level evidence.

---

## 23. Direct LAN Capture — PC2 Side

![Direct LAN PC2 capture](Screenshot/23-Direct-LAN-PC2-Capture.png)

This capture represents PC2 directly accessing the private IIS address.

Expected logical endpoints:

```text
Source      : 192.168.18.10
Destination : 192.168.18.100
Service     : TCP/80
```

### What this demonstrates

No VIP is needed for this test because both systems communicate through their private LAN addresses.

This gives a baseline TCP session to compare against NAT traffic.

---

## 24. Direct LAN Capture — IIS Server Side

![Direct LAN IIS server capture](Screenshot/24-Direct-LAN-IIS-Server-Capture.png)

The IIS-side capture confirms that the server received direct HTTP/TCP traffic from the internal LAN client.

Using captures at both endpoints helps verify that the expected packets are actually reaching the server.

---

## 25. External VIP Capture — External PC

![External VIP external PC capture](Screenshot/25-External-VIP-External-PC-Capture.png)

This capture represents the external client communicating with:

```text
172.18.2.11:80
```

From the external PC's perspective, the destination is the VIP address.

The client does not see `192.168.18.100` as its destination.

---

## 26. External VIP Capture — IIS Server

![External VIP IIS server capture](Screenshot/26-External-VIP-IIS-Server-Capture.png)

On the IIS side, the session arrives after FortiGate has performed destination translation.

This comparison demonstrates a key NAT concept:

```text
Client-side packet destination
        !=
Server-side translated destination
```

FortiGate modifies the relevant address information while maintaining the logical session.

---

## 27. Failed Hairpin Capture — SYN Retransmissions

Before the Hairpin configuration was working, PC2 showed TCP SYN retransmissions.

![Before Hairpin SYN retransmissions](Screenshot/27-Before-Hairpin-PC2-SYN-Retransmissions.png)

### What SYN retransmissions mean

The client attempted to start a TCP connection by sending a SYN.

When it did not receive the expected SYN-ACK response, it retransmitted the SYN.

Conceptually:

```text
PC2 → SYN → VIP
PC2 waits for SYN-ACK
No successful response
PC2 → SYN retransmission
```

This packet-level evidence matches the browser failure shown earlier.

It proves that the problem existed below the HTTP page itself—the TCP connection was not completing correctly.

---

## 28. Successful Hairpin Capture — PC2 Side

After the Hairpin policies were added, another capture was collected on PC2.

![After Hairpin PC2 capture](Screenshot/28-After-Hairpin-PC2-Capture.png)

The successful capture shows the TCP exchange occurring when PC2 accesses the VIP.

From PC2's perspective, it is communicating with:

```text
172.18.2.11
```

even though the real server remains:

```text
192.168.18.100
```

This abstraction is one of the main purposes of NAT.

---

## 29. Successful Hairpin Capture — IIS Server Side

The IIS-side packet capture was also checked after Hairpin NAT became operational.

![After Hairpin IIS server capture](Screenshot/29-After-Hairpin-IIS-Server-Capture.png)

This provides server-side evidence that the translated session successfully reached IIS.

Together, screenshots 28 and 29 show the same successful Hairpin connection from both sides of the session.

---

# Before vs After Hairpin NAT

| Test | Before Hairpin Policies | After Hairpin Policies |
|---|---|---|
| PC2 → IIS private IP | Works | Works |
| External PC → `172.18.2.11` | Works | Works |
| PC2 → `172.18.2.11` | Fails | Works |
| Browser result | Site cannot be reached | IIS page loads |
| TCP behavior | SYN retransmissions | Successful TCP exchange |
| Policy hits | Hairpin path not correctly matched | Hairpin policies used |
| Final result | FAIL | PASS |

---

# NAT Concepts Demonstrated

## Destination NAT

The VIP changes the destination:

```text
172.18.2.11
      ↓
192.168.18.100
```

This is the core destination NAT function used to publish the internal IIS server.

---

## Port Forwarding

Only TCP port 80 is published:

```text
172.18.2.11:80
      ↓
192.168.18.100:80
```

This is more specific than mapping every service on the address.

---

## Source NAT

The first Hairpin policy visibly has NAT enabled and uses the outgoing interface address.

Source NAT can be important in U-Turn designs because it helps ensure the server returns the traffic through the FortiGate rather than trying to bypass the firewall.

A complete NAT session must maintain a valid forward and return path.

---

## Hairpin / U-Turn NAT

Hairpin NAT occurs when a client located behind a firewall uses an address associated with the firewall's external side to access another host behind the same firewall.

In this lab:

```text
Internal client : 192.168.18.10
External VIP    : 172.18.2.11
Internal server : 192.168.18.100
```

The FortiGate receives the request from the LAN, translates it, and sends it back toward the LAN.

---

# Why Hairpin NAT Is Used in Real Networks

Hairpin NAT can be useful when internal and external users use the same published service address or name.

Example:

```text
Public DNS name:
portal.company.com
        ↓
Public/VIP address
203.0.113.10
```

External users reach the server through the public address.

Without split DNS, an internal user may resolve the same hostname to the same public/VIP address.

Hairpin NAT allows that internal user to still reach the internal server through the firewall.

---

# Troubleshooting Method Used

A useful Hairpin NAT troubleshooting sequence is:

## 1. Verify the server itself

```text
Is IIS running?
Can localhost open the page?
```

## 2. Verify the server IP and gateway

```text
Server IP      : 192.168.18.100
Default gateway: 192.168.18.1
```

## 3. Verify direct LAN access

```text
PC2 → 192.168.18.100
```

If this fails, fix the LAN problem before NAT.

## 4. Verify the VIP externally

```text
External PC → 172.18.2.11:80
```

If this fails, check VIP and WAN-to-LAN policy configuration.

## 5. Verify internal VIP access

```text
PC2 → 172.18.2.11:80
```

If direct access and external VIP access work but this fails, focus on Hairpin/U-Turn handling.

## 6. Check firewall policy hits

Confirm the expected Hairpin rules are being matched.

## 7. Check packet captures

Look for:

```text
SYN
SYN-ACK
ACK
Retransmissions
Translated source/destination addresses
```

## 8. Verify the return path

NAT is bidirectional session state. A request reaching the server is not sufficient if the reply does not return through the FortiGate correctly.

---

# Useful Verification Commands

## Windows

```powershell
ipconfig
ipconfig /all
ping 192.168.18.100
Test-NetConnection 192.168.18.100 -Port 80
Test-NetConnection 172.18.2.11 -Port 80
```

## Browser Tests

```text
Direct internal test:
http://192.168.18.100

VIP / Hairpin test:
http://172.18.2.11
```

## FortiGate CLI Commands Useful for Troubleshooting

The screenshots primarily document GUI configuration and Wireshark testing. During similar FortiGate troubleshooting, useful CLI checks include:

```bash
get system interface
show firewall vip
show firewall policy
get router info routing-table all
```

For flow troubleshooting, FortiGate debug flow can also be used carefully in a lab environment to identify policy and routing decisions.

---

# Test Summary

| Test | Expected Result | Result |
|---|---|---|
| IIS installation | IIS feature available | PASS |
| IIS localhost test | Web page loads | PASS |
| IIS private address | `192.168.18.100/24` | PASS |
| FortiGate LAN gateway | `192.168.18.1/24` | PASS |
| FortiGate WAN | `172.18.2.2/28` | PASS |
| VIP mapping | `172.18.2.11:80 → 192.168.18.100:80` | PASS |
| External PC addressing | `172.18.2.5/28` | PASS |
| External VIP browser test | IIS page loads | PASS |
| External VIP TCP/80 | `TcpTestSucceeded = True` | PASS |
| PC2 addressing | `192.168.18.10/24` | PASS |
| PC2 direct LAN ping | IIS reachable | PASS |
| PC2 direct TCP/80 | IIS HTTP port reachable | PASS |
| PC2 direct web access | IIS page loads | PASS |
| PC2 VIP test before Hairpin | Expected failure baseline | FAIL / EXPECTED |
| Hairpin policies configured | Required LAN/VIP path allowed | PASS |
| Hairpin policy hit verification | Matching traffic visible | PASS |
| PC2 VIP test after Hairpin | IIS page loads using `172.18.2.11` | PASS |
| Before-Hairpin packet capture | SYN retransmissions visible | PASS |
| After-Hairpin PC2 capture | Successful TCP conversation | PASS |
| After-Hairpin IIS capture | Translated session reaches server | PASS |

---

# Key Learning Outcomes

### 1. A VIP and a firewall policy perform different jobs

The VIP defines the translation.

The firewall policy authorizes the traffic.

A correct VIP without the required policy is not enough.

---

### 2. External VIP success does not prove Hairpin NAT success

The external PC successfully reached the IIS server through the VIP before PC2 could.

This proves that Hairpin NAT is a separate traffic case that must be tested independently.

---

### 3. Always prove direct connectivity first

PC2 was tested against `192.168.18.100` before testing the VIP.

This prevents basic LAN or IIS problems from being incorrectly diagnosed as NAT problems.

---

### 4. A controlled failure is valuable evidence

The screenshot showing `172.18.2.11` failing before Hairpin configuration is important because it demonstrates the actual problem that the later configuration solved.

---

### 5. Policy hit counters are an important troubleshooting tool

A policy that looks correct may not actually be selected.

Hit counters help verify real traffic matching.

---

### 6. Packet captures explain what the browser cannot

The browser only showed that the page failed.

Wireshark showed the lower-level symptom: TCP SYN retransmissions.

After the fix, the packet captures showed a successful TCP exchange.

---

### 7. NAT depends on the return path

The translated request and the reply must both pass through the FortiGate in a way that preserves the NAT session.

Incorrect return routing can break NAT even if the initial request reaches the server.

---

### 8. Hairpin NAT is essentially a U-turn through the firewall

The internal PC sends traffic toward an external/VIP address, but the translated destination is another internal host.

The FortiGate therefore sends the session back toward the LAN after processing it.

---

# Evidence Map

| Screenshot | Evidence |
|---|---|
| 01 | IIS Windows feature preparation |
| 02 | IIS server addressing and gateway |
| 03 | Default IIS web service works |
| 04 | Custom IIS page works locally |
| 05 | FortiGate WAN/LAN interface addressing |
| 06 | LAN software switch using port3 and port4 |
| 07 | VIP-side address object |
| 08 | External PC address object |
| 09 | LAN subnet address object |
| 10 | `VIP_IIS_HTTP` destination NAT and port forwarding |
| 11 | WAN-to-LAN IIS firewall policy |
| 12 | External PC `172.18.2.5/28` |
| 13 | External browser reaches VIP |
| 13A | External TCP port 80 succeeds |
| 14 | PC2 `192.168.18.10/24` CLI verification |
| 15 | PC2 GUI IP verification |
| 16 | Direct LAN ping and TCP test |
| 17 | Direct LAN browser access |
| 18 | Internal VIP access fails before Hairpin NAT |
| 19 | Hairpin LAN-to-WAN policy |
| 20 | Hairpin LAN-to-LAN VIP policy |
| 21 | Firewall policy list / hit counters |
| 22 | Hairpin VIP access succeeds |
| 23 | Direct LAN PC2 packet capture |
| 24 | Direct LAN IIS packet capture |
| 25 | External VIP client-side packet capture |
| 26 | External VIP IIS-side packet capture |
| 27 | Failed Hairpin SYN retransmissions |
| 28 | Successful Hairpin PC2 capture |
| 29 | Successful Hairpin IIS-side capture |

---

# Conclusion

This lab successfully demonstrated **FortiGate Hairpin NAT / U-Turn NAT** using an IIS web server.

The testing process established three clear states:

```text
Direct LAN access          → Working
External VIP access        → Working
Internal VIP before Hairpin → Failed
Internal VIP after Hairpin  → Working
```

The final result proves that the internal client `192.168.18.10` could access the internal IIS server `192.168.18.100` by using the external VIP `172.18.2.11`.

More importantly, the lab did not rely only on a successful browser screenshot. The configuration was validated through:

- Windows IP verification.
- IIS application testing.
- FortiGate VIP configuration.
- Firewall policy configuration.
- Policy hit counters.
- PowerShell TCP port testing.
- Direct connectivity testing.
- Before-and-after browser tests.
- Client-side Wireshark captures.
- Server-side Wireshark captures.
- SYN retransmission analysis.

This provides practical evidence of understanding **destination NAT, source NAT, VIPs, port forwarding, firewall policy matching, Hairpin/U-Turn NAT, return-path behavior, TCP connection establishment, and structured network troubleshooting**.
