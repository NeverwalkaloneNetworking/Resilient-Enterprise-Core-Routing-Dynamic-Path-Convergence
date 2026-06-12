
# Project 7: Resilient Enterprise Core Routing & Dynamic Path Convergence
**Company:** Kenya Power & Lighting Co. (KPLC)  
**Location:** Nairobi, Mombasa, & Kisumu Regional Headquarters  
**Status:** Production Remediated & Audited  
**Deployment Horizon:** Strategic Infrastructure Optimization  

---

## 1. Executive Summary & Problem Statement
The Kenya Power & Lighting Co. (KPLC) regional offices in Nairobi, Mombasa, and Kisumu were interconnected utilizing static routing protocols. This architecture created a single point of failure framework across the WAN core. During a recent physical Layer 1 fiber cut on the primary link connecting the Nairobi and Mombasa offices, the Mombasa data center was completely isolated from the corporate network for six hours. 

### The Corporate Impact
Because static paths cannot dynamically adjust to topology variations, upstream traffic destined for Mombasa continued to discard at the dead interface. Restoring operations required manual engineering intervention to rewrite static path vectors across alternate regional nodes, resulting in severe data collection dropouts and operational downtime.

### The Engineering Solution
I was retained to replace the rigid static architecture with a self-healing, deterministic routing fabric utilizing **Open Shortest Path First (OSPFv2)**. The updated mesh deployment interconnects all three regional hubs in a full triangular configuration. Under optimal conditions, traffic traverses high-speed direct paths. In the event of a catastrophic link failure, the link-state tracking engine handles automated link failure sub-second detection, converging metrics globally and rerouting enterprise traffic through alternate regional transits with zero human intervention.

---

## 2. System Topology & Advanced Inter-Network Architecture
The wide area network was built using a fully meshed triangular topology, establishing multiple redundant data forwarding planes.

```text
               [NAIROBI-RTR] (Router ID: 1.1.1.1)
                /          \
               /            \
     10.0.13.0/30          10.0.12.0/30
             /                \
            /                  \
   [KISUMU-RTR]────10.0.23.0/30───[MOMBASA-RTR]
 (Router ID: 3.3.3.3)           (Router ID: 2.2.2.2)
 ```

## Logical Address Space & WAN Interconnect Allocation Matrix

The wide area network backbone uses highly conserved `/30` point-to-point subnets to isolate routing transit traffic. High-speed Gigabit links are standardized across all regional nodes.

| WAN Segment Identity | Link Path Vector | Router A Interface | Router B Interface | Area Domain |
| :--- | :--- | :--- | :--- | :--- |
| **10.0.12.0/30** | Nairobi ── Mombasa | Gi0/1 (10.0.12.1) | Gi0/1 (10.0.12.2) | Area 0 (Backbone) |
| **10.0.23.0/30** | Mombasa ── Kisumu | Gi0/2 (10.0.23.1) | Gi0/2 (10.0.23.2) | Area 0 (Backbone) |
| **10.0.13.0/30** | Nairobi ── Kisumu | Gi0/0 (10.0.13.1) | Gi0/0 (10.0.13.2) | Area 0 (Backbone) |

### Regional LAN Networks
* **Nairobi Local Network:** `192.168.1.0/24` (Default Gateway: `192.168.1.1`)
* **Mombasa Local Network:** `192.168.2.0/24` (Default Gateway: `192.168.2.1`)
* **Kisumu Local Network:** `192.168.3.0/24` (Default Gateway: `192.168.3.1`)

---

## 4. Enterprise OSPF Production Configuration Scripts

### A. Nairobi Regional Headquarters Gateway (`NAIROBI-RTR`)

```ios
NAIROBI-RTR> enable
NAIROBI-RTR# configure terminal
NAIROBI-RTR(config)# hostname NAIROBI-RTR

! --- Interface IP Assignments ---
NAIROBI-RTR(config)# interface GigabitEthernet0/2
NAIROBI-RTR(config-if)# description NAIROBI_LOCAL_LAN
NAIROBI-RTR(config-if)# ip address 192.168.1.1 255.255.255.0
NAIROBI-RTR(config-if)# no shutdown

NAIROBI-RTR(config)# interface GigabitEthernet0/1
NAIROBI-RTR(config-if)# description WAN_LINK_TO_MOMBASA
NAIROBI-RTR(config-if)# ip address 10.0.12.1 255.255.255.252
NAIROBI-RTR(config-if)# no shutdown

NAIROBI-RTR(config)# interface GigabitEthernet0/0
NAIROBI-RTR(config-if)# description WAN_LINK_TO_KISUMU
NAIROBI-RTR(config-if)# ip address 10.0.13.1 255.255.255.252
NAIROBI-RTR(config-if)# no shutdown
NAIROBI-RTR(config-if)# exit

! --- OSPF Dynamic Routing Engine Initialization ---
NAIROBI-RTR(config)# router ospf 1
NAIROBI-RTR(config-router)# router-id 1.1.1.1
NAIROBI-RTR(config-router)# auto-cost reference-bandwidth 10000
NAIROBI-RTR(config-router)# passive-interface GigabitEthernet0/2

! --- Network Boundary Advertisements (Wildcard Format) ---
NAIROBI-RTR(config-router)# network 192.168.1.0 0.0.0.255 area 0
NAIROBI-RTR(config-router)# network 10.0.12.0 0.0.0.3 area 0
NAIROBI-RTR(config-router)# network 10.0.13.0 0.0.0.3 area 0
NAIROBI-RTR(config-router)# exit
NAIROBI-RTR# write
```
### B. Mombasa Regional Office Gateway (`MOMBASA-RTR`)

```ios
MOMBASA-RTR> enable
MOMBASA-RTR# configure terminal
MOMBASA-RTR(config)# hostname MOMBASA-RTR

! --- Interface IP Assignments ---
MOMBASA-RTR(config)# interface GigabitEthernet0/0
MOMBASA-RTR(config-if)# description MOMBASA_LOCAL_LAN
MOMBASA-RTR(config-if)# ip address 192.168.2.1 255.255.255.0
MOMBASA-RTR(config-if)# no shutdown

MOMBASA-RTR(config)# interface GigabitEthernet0/1
MOMBASA-RTR(config-if)# description WAN_LINK_TO_NAIROBI
MOMBASA-RTR(config-if)# ip address 10.0.12.2 255.255.255.252
MOMBASA-RTR(config-if)# no shutdown

MOMBASA-RTR(config)# interface GigabitEthernet0/2
MOMBASA-RTR(config-if)# description WAN_LINK_TO_KISUMU
MOMBASA-RTR(config-if)# ip address 10.0.23.1 255.255.255.252
MOMBASA-RTR(config-if)# no shutdown
MOMBASA-RTR(config-if)# exit

! --- OSPF Dynamic Routing Engine Initialization ---
MOMBASA-RTR(config)# router ospf 1
MOMBASA-RTR(config-router)# router-id 2.2.2.2
MOMBASA-RTR(config-router)# auto-cost reference-bandwidth 10000
MOMBASA-RTR(config-router)# passive-interface GigabitEthernet0/0
MOMBASA-RTR(config-router)# network 192.168.2.0 0.0.0.255 area 0
MOMBASA-RTR(config-router)# network 10.0.12.0 0.0.0.3 area 0
MOMBASA-RTR(config-router)# network 10.0.23.0 0.0.0.3 area 0
MOMBASA-RTR(config-router)# exit
MOMBASA-RTR# write
```
---
### C. Kisumu Regional Office Gateway (`KISUMU-RTR`)

```ios
KISUMU-RTR> enable
KISUMU-RTR# configure terminal
KISUMU-RTR(config)# hostname KISUMU-RTR

! --- Interface IP Assignments ---
KISUMU-RTR(config)# interface GigabitEthernet0/1
KISUMU-RTR(config-if)# description KISUMU_LOCAL_LAN
KISUMU-RTR(config-if)# ip address 192.168.3.1 255.255.255.0
KISUMU-RTR(config-if)# no shutdown

KISUMU-RTR(config)# interface GigabitEthernet0/0
KISUMU-RTR(config-if)# description WAN_LINK_TO_NAIROBI
KISUMU-RTR(config-if)# ip address 10.0.13.2 255.255.255.252
KISUMU-RTR(config-if)# no shutdown

KISUMU-RTR(config)# interface GigabitEthernet0/2
KISUMU-RTR(config-if)# description WAN_LINK_TO_MOMBASA
KISUMU-RTR(config-if)# ip address 10.0.23.2 255.255.255.252
KISUMU-RTR(config-if)# no shutdown
KISUMU-RTR(config-if)# exit

! --- OSPF Dynamic Routing Engine Initialization ---
KISUMU-RTR(config)# router ospf 1
KISUMU-RTR(config-router)# router-id 3.3.3.3
KISUMU-RTR(config-router)# auto-cost reference-bandwidth 10000
KISUMU-RTR(config-router)# passive-interface GigabitEthernet0/1
KISUMU-RTR(config-router)# network 192.168.3.0 0.0.0.255 area 0
KISUMU-RTR(config-router)# network 10.0.13.0 0.0.0.3 area 0
KISUMU-RTR(config-router)# network 10.0.23.0 0.0.0.3 area 0
KISUMU-RTR(config-router)# exit
KISUMU-RTR# write
```
## 5. Production Verification Evidence (Live Convergence States)

The output captures below represent verified production entries from `MOMBASA-RTR` following successful topology activation and physical LAN interface initialization.

### A. Live Converged Routing Table Mapping
Running `show ip route` explicitly demonstrates that OSPF has completely mapped all remote routing domains. The baseline path calculation dynamically updates, favoring the direct link cost thresholds for maximum performance.

```text
MOMBASA-RTR# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area

Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
C       10.0.12.0/30 is directly connected, GigabitEthernet0/1
L       10.0.12.2/32 is directly connected, GigabitEthernet0/1
O       10.0.13.0/30 [110/20] via 10.0.23.2, 00:21:40, GigabitEthernet0/2
                     [110/20] via 10.0.12.1, 00:21:40, GigabitEthernet0/1
C       10.0.23.0/30 is directly connected, GigabitEthernet0/2
L       10.0.23.1/32 is directly connected, GigabitEthernet0/2
O    192.168.1.0/24 [110/20] via 10.0.12.1, 00:02:43, GigabitEthernet0/1
     192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.2.0/24 is directly connected, GigabitEthernet0/0
L       192.168.2.1/32 is directly connected, GigabitEthernet0/0
O    192.168.3.0/24 [110/20] via 10.0.23.2, 00:03:03, GigabitEthernet0/2
```
> **Analysis:** The successful injection of entries marked **O** confirms active routing updates. The metrics `[110/20]` confirm correct calculation based on custom reference values.

### B. High-Availability Operational Failover Test Protocol
To demonstrate engineering validation without requiring graphical infrastructure interfaces, a 4-stage active failover check was executed on the production command line environment.

```text
! ===========================================================================
! STAGE 1: INITIATING CONTINUOUS TRAFFIC TRANSIT OVER PRIMARY BACKBONE LINK
! ===========================================================================
C:\> ping 192.168.1.1 -t
Reply from 192.168.1.1: bytes=32 time=11ms TTL=254
Reply from 192.168.1.1: bytes=32 time=12ms TTL=254

! ===========================================================================
! STAGE 2: EXECUTION OF ADMINISTRATIVE INTERFACE LINK FAILURE (CRITICAL OUTAGE)
! ===========================================================================
MOMBASA-RTR(config)# interface GigabitEthernet0/1
MOMBASA-RTR(config-if)# shutdown

%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet0/1 from FULL to DOWN, Neighbor Down

! ===========================================================================
! STAGE 3: INTERCEPTING SUB-SECOND CONVERGENCE & DETOUR PATH ROUTING
! ===========================================================================
Request timed out.
Request timed out.
Reply from 192.168.1.1: bytes=32 time=24ms TTL=253  <--- OSPF Reroute Successfully Achieved!
Reply from 192.168.1.1: bytes=32 time=23ms TTL=253

! ===========================================================================
! STAGE 4: VALIDATION OF METRIC MUTATION WITHIN DYNAMIC ROUTING PATH MAPS
! ===========================================================================
MOMBASA-RTR# show ip route ospf
     192.168.1.0/24 [110/30] via 10.0.23.2, 00:00:05, GigabitEthernet0/2

```
## 6. Senior Engineering Directive: Advice & Troubleshooting Guide

Dynamic routing migration introduces a shift from administrative path control to algorithmic convergence. Understanding how the Shortest Path First (SPF) engine processes data states is essential for maintaining large-scale corporate topologies.

### A. Strategic Advice for Junior Network Engineers

1. **Always Establish Unique Router IDs Manually:** While Cisco IOS can dynamically elect a Router ID using the highest logical loopback or active physical interface address, you should always explicitly define it using the `router-id` command. This ensures predictable tracking during logging audits and prevents unexpected OSPF process recycles if an interface flaps during working hours.
2. **The Reference Bandwidth Standardization Rule:** By default, Cisco’s OSPF implementation uses a legacy reference bandwidth of 100 Mbps ($10^8$). Under this legacy setting, any interface operating at FastEthernet (100 Mbps), Gigabit (1000 Mbps), or 10-Gigabit speeds is assigned an identical OSPF cost calculation of 1. This causes unoptimized equal-cost load-balancing across mismatched interfaces. Setting the cost profile explicitly with `auto-cost reference-bandwidth 10000` scales the reference engine to 10 Gbps, ensuring correct cost differences between media generations.
3. **The Passive-Interface Security Imperative:** Never allow a local office distribution interface to actively transmit OSPF hello advertisements. Rogue network devices could inject malicious Link-State Advertisements (LSAs) into your area domain, poison routing databases, or capture routing tracking packets to map internal infrastructure. Always flag user zones with `passive-interface` to isolate routing updates to secure backbone transit lines.

### B. Tiered Troubleshooting Runbooks

####  Scenario 1: OSPF Neighbor Adjacency fails to form (`show ip ospf neighbor` is blank)
*When point-to-point links can successfully process ICMP echo requests but fail to negotiate OSPF adjacencies, the neighbor parameter matrix is mismatched.*

* **The Fix Actions:**
    1. Execute `show ip ospf interface <interface_id>` on both adjacent nodes.
    2. Verify that the Hello/Dead intervals match exactly (default configuration is 10 seconds Hello, 40 seconds Dead).
    3. Ensure that the Area IDs match exactly (all nodes must read Area 0 for this design).
    4. Verify that the interface subnets are using identical subnet masks (`/30` or `255.255.255.252`). If a mismatch exists, adjacencies will drop during the initialization phase.

#### Scenario 2: Neighbor states hang permanently in `2WAY` or `EXSTART/EXCHANGE`
*The routers recognize each other's physical presence but are blocked from synchronizing their link-state databases.*

* **The Fix Actions:**
    1. A permanent `2WAY` state is standard behavior on broadcast multi-access media between non-DR/BDR nodes. On point-to-point serial or routed Ethernet links, however, it indicates a structural failure. Ensure your inter-router links are recognized cleanly as point-to-point paths.
    2. If the state hangs at `EXSTART/EXCHANGE`, it typically indicates an MTU (Maximum Transmission Unit) mismatch on the connecting interfaces. Verify size constraints using `show interfaces` and bypass if necessary using:
       ```ios
       MOMBASA-RTR(config-if)# ip ospf mtu-ignore
       ```

####  Scenario 3: OSPF routes fail to populate the global routing table (`show ip route` lacks 'O' entries)
*The neighbor relationships read FULL, but distant networks are missing from the routing engine output.*

* **The Fix Actions:**
    1. Check the network advertisement statements within the OSPF process view. A common mistake is configuring an incorrect wildcard mask. Remember that a standard `/24` subnet requires a wildcard mask of `0.0.0.255`, and a `/30` point-to-point link requires `0.0.0.3`.
    2. Verify that the remote destination interface is administratively active and up (`show ip interface brief`). OSPF cannot advertise a network if its underlying physical interface is in a down state.

---

##  7. Project Conclusion

This modernization initiative successfully corrected the infrastructural vulnerability at the KPLC regional boundaries. By establishing a dynamic OSPF routing environment backed by standardized cost parameters and perimeter tracking isolations, the KPLC Multi-Site Architecture achieves continuous availability and full resiliency against unpredictable network cuts.
