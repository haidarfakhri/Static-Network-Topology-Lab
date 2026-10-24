# Network Topology Analysis Report

**Date:** October 24, 2026;
**Subject:** Comparative Analysis of Direct vs. Intermediate Router Static Topologies

---

## 1. Executive Summary

This report details the analysis of two distinct static network topologies involving two primary subnets (**192.168.1.0/24** and **192.168.2.0/24**). 

*   **Topology A** utilizes a direct connection between Router 1 and Router 2 over a single point-to-point link.
*   **Topology B** introduces a third intermediary router (Router 3), creating a multi-hop path between the two end networks.

Both topologies rely on static routing configuration. This analysis covers IP addressing schemes, routing table requirements, connectivity paths, and operational implications for each design.

---

## 2. Topology A: Direct Router Interconnection

In this scenario, Router 1 and Router 2 are directly linked via a serial or crossover Ethernet link using the `192.168.3.0/24` subnet.

### 2.1. Device Inventory & IP Schema

| Device | Role | Interface | IP Address / Mask | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **Host A1** | End User | NIC | 192.168.1.2/24 | 192.168.1.1 |
| **Host A2** | End User | NIC | 192.168.1.3/24 | 192.168.1.1 |
| **Host A3** | End User | NIC | 192.168.1.4/24 | 192.168.1.1 |
| **Switch 1** | L2 Access | N/A | N/A | N/A |
| **Router 1** | Edge GW | G0/0 (LAN) | 192.168.1.1/24 | N/A |
| **Router 1** | Edge GW | G0/1 (WAN) | **192.168.3.1/24** | N/A |
| **Router 2** | Edge GW | G0/0 (LAN) | 192.168.2.1/24 | N/A |
| **Router 2** | Edge GW | G0/1 (WAN) | **192.168.3.2/24** | N/A |
| **Host B1** | End User | NIC | 192.168.2.2/24 | 192.168.2.1 |
| **Host B2** | End User | NIC | 192.168.2.3/24 | 192.168.2.1 |
| **Switch 2** | L2 Access | N/A | N/A | N/A |

### 2.2. Connectivity & Routing Logic

To allow communication between Subnet A and Subnet B, Static Routes must be configured on both edge routers. Traffic flows linearly.

#### Required Static Routes
1.  **On Router 1:** Needs a path to Destination Network 2.
    ```text
    ip route 192.168.2.0 255.255.255.0 192.168.3.2
    ```
2.  **On Router 2:** Needs a path to Destination Network 1.
    ```text
    ip route 192.168.1.0 255.255.255.0 192.168.3.1
    ```

#### Data Path Example (Host A1 to Host B1)
1.  **Source:** Host A1 checks Gateway. Dest (192.168.2.2) != Local. Sends to GW (192.168.1.1).
2.  **R1:** Receives packet. Lookup Table finds route `192.168.2.0/24` via Next Hop `192.168.3.2`. Forwards to R2.
3.  **R2:** Receives packet. Lookup Table finds interface `192.168.2.1` (Connected). ARP resolves Host B1 MAC.
4.  **Dest:** Host B1 receives packet. Reply follows reverse path.

### 2.3. Analysis of Topology A
*   **Efficiency:** High. Single hop between routers reduces latency and CPU load on intermediate devices.
*   **Failure Domain:** Low redundancy. If the cable/link between R1 and R2 fails, total inter-subnet communication stops. There is no failover without additional protocols (OSPF/HSRP).
*   **Complexity:** Low. Easy to troubleshoot; only two routers manage the routing state.
*   **Subnet Usage:** One dedicated transit subnet (`192.168.3.0/24`). Efficient use of IPv4 space.

---

## 3. Topology B: Multi-Hop via Intermediate Router (Router 3)

In this scenario, the direct link is broken, and Router 3 sits between Router 1 and Router 2. This creates two distinct transit segments (`192.168.3.0/24` and `192.168.4.0/24`).

### 3.1. Device Inventory & IP Schema
*Based on the provided Hop/IP assignments.*

| Device | Interface Link | Assigned IP | Role / Next Hop Context |
| :--- | :--- | :--- | :--- |
| **Router 1** | G0/0 (to R3) | **192.168.3.2/24** | Endpoint for Segment 3 |
| **Router 3** | East Port (to R1) | **192.168.3.1/24** | Next Hop for R1 |
| **Router 3** | West Port (to R2)| **192.168.4.2/24** | Next Hop for R2 |
| **Router 2** | G0/1 (to R3) | **192.168.4.1/24** | Endpoint for Segment 4 |
| *(Note)* | *(Other Interfaces)* | *(See Topology A for LAN)* | *(LAN IPs remain unchanged)* |

*Note: Hosts on switches maintain original IPs and default gateways.*

### 3.2. Connectivity & Routing Logic

This topology requires "hopping." Router 3 acts as a relay. All three routers require static routing entries.

#### Required Static Routes
1.  **On Router 1 (Source):** Needs to reach Network 2.
    *   Destination: `192.168.2.0/24`
    *   Next Hop: `192.168.3.1` (R3 Interface)
    ```text
    ip route 192.168.2.0 255.255.255.0 192.168.3.1
    ```
2.  **On Router 3 (Transit):** Needs to know how to return traffic to Network 1 AND forward traffic to Network 2.
    *   To Net 1: Next Hop `192.168.3.2` (R1)
    *   To Net 2: Next Hop `192.168.4.1` (R2)
    ```text
    ip route 192.168.1.0 255.255.255.0 192.168.3.2
    ip route 192.168.2.0 255.255.255.0 192.168.4.1
    ```
3.  **On Router 2 (Destination):** Needs to reach Network 1.
    *   Destination: `192.168.1.0/24`
    *   Next Hop: `192.168.4.2` (R3 Interface)
    ```text
    ip route 192.168.1.0 255.255.255.0 192.168.4.2
    ```

### 3.3. Analysis of Topology B
*   **Efficiency:** Reduced. Packets must be processed twice (R1 -> R3 -> R2) instead of once (R1 -> R2). Increased latency due to extra store-and-forward operations.
*   **Scalability:** Improved for hierarchy. R3 can act as a regional distribution layer if more sites are added later (branch office extension).
*   **Security Boundary:** R3 can implement stricter Access Control Lists (ACLs) between the regions, acting as a firewall/chokepoint before reaching the core (if R1/R2 were cores).
*   **Point of Failure:** **Increased.** Two links (R1-R3 and R3-R2) and three routers must be operational for full connectivity. If R3 goes down, the entire network splits in two.
*   **Administrative Overhead:** Higher. You must manage, update, and secure an additional device (Router 3).

---

## 4. Comparative Analysis

| Feature | Topology A (Direct) | Topology B (Via R3) |
| :--- | :--- | :--- |
| **Path Length** | 1 Hop (R1 → R2) | 2 Hops (R1 → R3 → R2) |
| **Routing Config** | 2 Static Routes total | 4 Static Routes total (R3 adds 2) |
| **Subnets Used** | 3 Total (Net 1, Net 2, Transit) | 4 Total (Net 1, Net 2, Transit A, Transit B) |
| **Reliability** | Single Point of Failure (Link R1-R2) | Triple Point of Failure (R3, Link A, Link B) |
| **Latency** | Lower | Higher (Extra processing delay) |
| **Use Case** | Small branch connections, simple WANs | Hierarchical networks, security segmentation |

### 4.1. Potential Configuration Issues Identified

1.  **IP Address Confusion in Prompt (Topology B):**
    *   In the prompt, R1 G0/0 is `.3.2` while the "R1 Hop" is `.3.1`.
    *   *Risk:* Admins may confuse the "Local Interface IP" with the "Next Hop IP." As documented above, R1 sends to R3 (`3.1`), so R1 must own `3.2`. The configuration must be explicit to avoid neighbor adjacency failures.
2.  **Return Traffic:**
    *   A common error in static networking is configuring the outbound route but forgetting the return route on the destination router (or R3 in Topo B). If R3 knows where to send traffic to Net 2, but doesn't have a route back to Net 1, pings will fail one-way.
3.  **Host Gateway Settings:**
    *   Hosts (A1-B2) must have their Default Gateway set strictly to their respective Router LAN Interface (`.1` addresses). If Host A1 sets its GW to Router 1's *WAN* IP or an invalid address, traffic never leaves the switch.
4.  **Subnet Mask Consistency:**
    *   All subnets are `/24`. In production, ensure the interface mask matches the configured mask exactly. Mismatched masks (e.g., R1 says `/24`, R3 says `/25`) will cause connectivity loss even if IP addresses fall within range.

---

## 5. Recommendations

1.  **Documentation:** Maintain a current Layer 3 diagram showing IP assignment per interface. Do not rely solely on "Next Hop" labels; document actual interface IPs (e.g., R1 G0/0 = x.x.x.x).
2.  **Redundancy Strategy:**
    *   If choosing **Topology B**, consider enabling a dynamic routing protocol (like OSPFv2) instead of static routing. This would automatically recalculate routes if Link R1-R3 fails and reroute via another path if available.
    *   If sticking to **Static**, implement Floating Static Routes with higher administrative distance for backup links.
3.  **Troubleshooting Order:**
    *   **Step 1:** Verify Physical Layer (Cable lights on Switches/Routers).
    *   **Step 2:** Verify Local Connectivity (Host Pings Gateway).
    *   **Step 3:** Verify Hop-to-Hop Connectivity (Ping R1 to R3, then R3 to R2).
    *   **Step 4:** Verify Routing Tables (`show ip route`).
4.  **Topology Choice:**
    *   For small networks (<50 nodes), **Topology A** is superior due to simplicity and performance.
    *   If there is a requirement for policy enforcement between the two branches, **Topology B** allows placing a firewall or ACL-heavy router at R3 location.

## 6. Conclusion

Both topologies are functionally valid static routing environments. **Topology A** represents the industry-standard approach for simple site-to-site connectivity, offering lower complexity and better performance. **Topology B** introduces architectural complexity that may be justified for hierarchical growth or specific security segmentation, but it increases the risk of outages due to the introduction of an additional single point of failure (Router 3).

Given the constraints provided, **Topology A** is recommended for initial deployment unless a specific requirement for the intermediate Router 3 exists (e.g., bandwidth throttling, logging, or physical distance requiring signal regeneration).

---
**End of Report**
