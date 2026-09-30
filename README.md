# Project Zero Day: Enterprise Network Hardening & Core Lab Infrastructure

## 1. Executive Summary & Strategic Objectives

### 1.1 Intent and Vision
Project "Zero Day" (ZDOG) marks the systematic engineering of an isolated, multi-tier, defense-in-depth cybersecurity laboratory designed for Security Operations Center (SOC) simulation, live telemetry aggregation, active threat hunting, and malware sandbox isolation. This repository serves as the definitive technical case study and architectural blueprint for the Pre-Ubuntu baseline phase of the infrastructure.

By moving away from flat, consumer-grade topologies, this architecture implements absolute logical isolation and rigorous access control planes. This layout ensures that high-risk laboratory environments can coexist safely alongside standard production data segments without cross-contamination.

'''
┌──────────────────────────────────────────────────────────────────────────┐
│                        PROJECT DESIGN BLUEPRINT                          │
├───────────────────┬──────────────────────────────┬───────────────────────┤
│    FRAMEWORK      │     CREDENTIAL ALIGNMENT     │  COLLABORATIVE EDGE   │
├───────────────────┼──────────────────────────────┼───────────────────────┤
│ Defense-in-Depth  │ • CompTIA Network+           │ • Human-AI Co-Ops     │
│ Zero Trust Core   │ • ISC2 CC                    │ • Automated Linting   │
│ Telemetry Hook    │ • Microsoft SC-900           │ • Syntax Validation   │
└───────────────────┴──────────────────────────────┴───────────────────────┘
'''


### 1.2 Human-AI Collaborative Engineering
In alignment with modern operational efficiency, advanced Artificial Intelligence (AI) was leveraged as a core collaborative mechanism throughout this initiative. The AI functioned as a technical force-multiplier—accelerating troubleshooting lifecycles, validating kernel network daemon configurations, and structuring enterprise-grade documentation layouts.

---

## 2. Hardware Specifications & Resource Baseline

The physical and logical layers of the aggregation fabric are strictly cataloged below to maintain absolute asset integrity.

### 2.1 Network & Switching Topology
* **Broadband Termination Gateway:** Xfinity XB8 Wi-Fi 6E Gateway (Provisioned strictly in transparent Bridge Mode to completely eliminate upstream carrier NAT layers).
* **Perimeter Stateful Router:** TP-Link Archer AX21 (AX1800 Dual-Band Wi-Fi 6 Router). Manages primary WAN state tables and edge NAT translations.
* **Layer 2 Managed Switching Element:** Netgear GS308E 8-Port Gigabit Smart Managed Plus Switch. Serves as the primary hardware enforcement engine for 802.1Q encapsulation, frame tagging, and PVID mapping.
* **Downstream Port Collector:** Netgear GS316 16-Port Gigabit Unmanaged Switch. Utilized strictly for the physical extension of the isolated Cyber Lab network segment.

### 2.2 Core Infrastructure Node
* **Physical Host Server:** Lenovo Small Form Factor (SFF) Hardware Platform.
* **Operating System:** Fedora Server (Bare-Metal Deployment).
* **Primary System Role:** Serving as the central computational engine for security telemetry aggregation, packet capture processing, and running the internal monitoring stacks (Wazuh Manager and the Elastic/ELK log indexing stack).
* **Physical Interconnect:** Connected via a dedicated Cat6 run into the Netgear switching matrix, dynamically reconfigured from an access link to an active 802.1Q sub-interfaced trunk link.

---

## 3. Phase 1: WAN Stabilization & Transport Layer Remediation

### 3.1 Resolving Layer 1 Signal Noise Anomalies
During initial environment stability testing, the edge gateway experienced chronic synchronization failures and unprovoked hardware reboots. 

An inspection of the local syslog and modem diagnostic interfaces revealed that upstream power levels were peaking at **54 dB**. This severely exceeded the maximum DOCSIS operational ceiling of 52 dB, causing the cable modem's internal logic to drop its connection with the Cable Modem Termination System (CMTS).

**Resolution:** A physical layer audit was performed across the ingress cabling. Extraneous coaxial splitters were removed, terminal connections were replaced, and physical interface vectors were tightened. This physical remediation reduced line noise by **3.5 dB**, dropping the return path back into safe tolerance and permanently resolving the synchronization drops.

### 3.2 Eradicating the Layer 3 Ghost DHCP Conflict
When the Xfinity XB8 gateway was toggled into transparent Bridge Mode, the primary TP-Link Archer AX21 router failed to lease a public WAN address. Instead, the network entered a highly unstable, unrouted "Double NAT" state.

Diagnostic syslogs captured continuous `DHCP DISCOVER` timeouts followed by unexpected local lease offers:

dhcpc: <6> send discover with ip 0.0.0.0
dhcpc: <4> timeout waiting for dhcp offer
dhcpc: <6> send discover with ip 0.0.0.0
dhcpc: <6> receive offer from server with ip 10.0.0.1

Root Cause Analysis: Even when configured for Bridge Mode, the consumer-grade carrier firmware left an integrated, hidden DHCPv4 server active on the local subnet 10.0.0.1/24. This was traced directly to the active broadcast of the unauthenticated Xfinity Public Wi-Fi Hotspot mesh network. This rogue management stack intercepted the TP-Link router’s initialization routines, preventing it from obtaining a public IP lease.

Resolution: A full factory hardware reset was applied to the gateway node. The core Xfinity Account Portal back-end systems were then used to issue a manual administrative command to suppress the public hotspot SSID broadcast, permanently disabling the rogue 10.0.0.1 management daemon.

###3.3 Outbound WAN Optimization
To optimize performance and minimize handshake delays during strict 1-hour ISP lease renewal cycles, two critical adjustments were enforced on the TP-Link Archer AX21 router:

MAC Address Cloning: Forced the ISP head-end to permanently bind the public lease allocation to a consistent hardware signature.

MTU Optimization: Locked the Maximum Transmission Unit to exactly 1472 bytes (+28 bytes for IP/ICMP headers to fit the standard 1500 frame boundary), eliminating packet fragmentation across the WAN boundary.

On May 7, 2026, at 17:34:57, the network achieved a stable, public IP allocation via a clean, sub-second DHCP ACK transition, establishing a 99.9% network uptime baseline:

[2026-05-08 09:33:50] dhcpc: <6> 1/2 lease passed, enter renewing state
[2026-05-08 10:33:54] dhcpc: <6> send select request (cliid=01/50:3d:d1:xx:xx:xx)
[2026-05-08 10:33:55] dhcpc: <6> receive ack from server with ip 73.82.99.123

## 4. Phase 2: Tiered Network Segmentation (Router-on-a-Switch Model)

To isolate laboratory vulnerability assessments from standard household traffic, a 3-tier defense-in-depth logical topology was designed and deployed.

### 4.1 Logical Network Allocation Matrix

'''
┌────────────────────────────────────────────────────────────────────────┐
│                      VLAN SEGMENTATION PROFILE                         │
├─────────┬───────────────────┬───────────────────┬──────────────────────┤
│ VLAN ID │    TIER NAME      │  SUBNET BOUNDARY  │   SECURITY PROFILE   │
├─────────┼───────────────────┼───────────────────┼──────────────────────┤
│   10    │ Home / Trusted    │ 192.168.0.0/24    │ Strict Internal AP   │
│   20    │ IoT Isolated      │ 192.168.20.0/24   │ Air-Gapped Sandbox   │
│   50    │ ZDOG Cyber Lab    │ 172.16.0.0/24     │ Full Telemetry Tap   │
└─────────┴───────────────────┴───────────────────┴──────────────────────┘
'''

### 4.2 Switch-Level Hardening on the Netgear GS308E
To protect against Layer 2 reconnaissance, spoofing, and lateral data injection, the 802.1Q advanced configuration pages on the managed switch were manually hardened:

Parking VLAN 1 (VLAN Hopping Mitigation): To completely eliminate default VLAN reconnaissance or trunk-spoofing injection vectors, the native factory-default VLAN 1 was entirely "parked." Every port on the GS308E (except for the Port 1 Uplink trunk) was explicitly marked as Excluded (E) from VLAN 1 membership.

Port Tagging & Membership Layout:
Port 1 (Uplink Trunk to AX21): Tagged (T) for VLANs 10, 20, and 50. PVID: 1.
Port 2 (IoT AP Link): Untagged (U) for VLAN 20; Excluded from 10 and 50. PVID: 20.
Port 3 (Lab Extension Switch - GS316): Untagged (U) for VLAN 50; Excluded from 10 and 20. PVID: 50. (Forces untagged incoming traffic from the 16-port extension switch into the 172.16.0.0/24 subnet).
Ports 4-8 (Domestic Endpoints): Untagged (U) for VLAN 10; Excluded from 20 and 50. PVID: 10.

## 5. Phase 3: The Architectural Pivot to Fedora "Router-on-a-Stick"

### 5.1 Tactical Advantage of Server-Centric Routing
While the initial network layout effectively isolated traffic, the consumer-grade firmware on the TP-Link router abstracted the underlying data plane. This configuration introduced a critical blind spot for security engineering and log analysis.
The architecture was intentionally modified to implement a Router-on-a-Stick topology via the Bare-Metal Fedora Server. By moving the logical routing intelligence onto the open Linux server kernel, the Fedora host was transformed into the primary internal firewall and traffic controller for the lab.

This approach yielded a significant tactical advantage: it allowed network monitoring tools to bind directly to the virtual sub-interfaces (such as enp3s0.50). When an internal machine inside the Cyber Lab initiates a network connection, the packets must pass through the Fedora kernel to be routed. This creates a high-fidelity vantage point to analyze raw traffic patterns and intercept lateral movement before the data ever reaches an external internet gateway.

                  ┌────────────────────────┐
                  │   TP-Link Archer WAN   │
                  └───────────┬────────────┘
                              │ (Untagged Uplink)
                  ┌───────────┴────────────┐
                  │ Netgear GS308E Switch  │
                  └───────────┬────────────┘
                              │ (802.1Q Trunk Link)
                  ┌───────────┴────────────┐
                  │   Fedora Server Host   │
                  │  Interface: enp3s0     │
                  │  ├──────────────────┤  │
                  │  │    enp3s0.10     │  │ (Gateway: 192.168.0.1)
                  │  ├──────────────────┤  │
                  │  │    enp3s0.20     │  │ (Gateway: 192.168.20.1)
                  │  ├──────────────────┤  │
                  │  │    enp3s0.50     │  │ (Gateway: 172.16.0.1)
                  │  └──────────────────┘  │
                  └────────────────────────┘

### 5.2 Step-by-Step Low-Level Linux Implementation
**Step 1:** Kernel 802.1Q Driver Initialization
To enable the OS to process tagged Ethernet headers, the appropriate driver module was loaded into kernel space:

> sudo modprobe 8021q

To ensure the module remains persistent across system reboots, the configuration was hardcoded to the system startup load files:

> sudo modprobe 8021q

To ensure the module remains persistent across system reboots, the configuration was hardcoded to the system startup load files:

> echo "8021q" | sudo tee /etc/modules-load.d/8021q.conf

**Step 2:** Virtual Tagged Sub-Interface Architecture
Using the NetworkManager CLI (nmcli), explicit virtual sub-interfaces were bound to the physical network interface card (enp3s0) to listen for the switch tags.

Deploying the Cyber Lab Default Gateway (VLAN 50):

> sudo nmcli con add type vlan con-name vlan50 ifname enp3s0.50 dev enp3s0 id 50
> sudo nmcli con mod vlan50 ipv4.addresses 172.16.0.1/24 ipv4.method manual
> sudo nmcli con up vlan50

Deploying the IoT Sandbox Default Gateway (VLAN 20):

> sudo nmcli con add type vlan con-name vlan20 ifname enp3s0.20 dev enp3s0 id 20
> sudo nmcli con mod vlan20 ipv4.addresses 192.168.20.1/24 ipv4.method manual
> sudo nmcli con up vlan20

**Step 3:** Enabling Kernel Packet Forwarding
By default, standard Linux server distributions drop transit packets not explicitly addressed to a local host socket. To enable active Layer 3 routing functionality, the kernel data plane was configured to forward transit packets:

echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ipforward.conf
> sudo sysctl -p /etc/sysctl.d/99-ipforward.conf

**Step 4:** Firewall Zone Engineering and Unidirectional ACL Design
With routing enabled, the system would natively bridge all zones together without restriction. To enforce strict security boundaries, the firewalld daemon was configured as a stateful access control mechanism, ensuring that high-risk lab networks remained completely isolated from production devices.

* **1. Bind the newly provisioned virtual sub-interfaces to explicit security zones**
> sudo firewall-cmd --permanent --zone=internal --add-interface=enp3s0.50
> sudo firewall-cmd --permanent --zone=internal --add-interface=enp3s0.20

* **2. Establish the primary physical untagged interface as the untrusted WAN uplink**
> sudo firewall-cmd --permanent --zone=external --add-interface=enp3s0

* **3. Enforce IP Masquerading (NAT) across the external egress interface.**
*This allows virtual lab nodes to securely download external security updates
without exposing their private internal IP addresses to the home network.*
> sudo firewall-cmd --permanent --zone=external --add-masquerade

* **4. Flush and reload the firewalld runtime environment to execute policy blocks**
> sudo firewall-cmd --reload


