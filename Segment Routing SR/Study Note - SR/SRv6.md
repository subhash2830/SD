


---

## 1. What is SRv6?

**Simple answer**  
SRv6 is **Segment Routing over IPv6**. The ingress router decides the end‑to‑end path and encodes it as a list of IPv6 addresses (segments) inside the packet, so core routers do not need per‑flow state.

**Deep dive**  
SRv6 uses the **IPv6 data plane** instead of MPLS labels: each segment is a 128‑bit IPv6 address called an SRv6 SID. The segment list is carried in the **Segment Routing Header (SRH)**, an IPv6 extension header defined in RFC 8754. SRv6 follows the same principles as SR‑MPLS but extends them with **network programming**, where SIDs can represent functions, not just locations.[web:52][web:55]

---

## 2. What is an SRv6 SID and how is it structured?

**Simple answer**  
An SRv6 SID is an IPv6 address that tells the network both **where to send the packet** and **what to do with it**. It is more than a normal IPv6 address because it encodes a function.

**Deep dive**  
Logically, a SID is split into three fields: **Locator**, **Function**, and optional **Arguments**.[web:48][web:55] The locator is a routable IPv6 prefix that gets the packet to the correct node. The function identifies what that node should do with the packet (for example, act as a VPN endpoint or adjacency). Arguments can carry additional context such as VRF IDs or service identifiers. This allows a single SID to encode both **transport** and **service** behavior.[web:55]

---

## 3. What is the SRv6 Locator?

**Simple answer**  
The **locator** is the routable part of the SID. It behaves like a normal IPv6 prefix and is used by the IGP to forward the packet to the right node.

**Deep dive**  
Operators typically allocate a locator per node or per role (for example, per PE or per data‑center block), such as `2001:db8:10:1::/64`.[web:55] Any SID starting with that locator will reach the same node, and the remaining bits (function + arguments) are interpreted locally. This design lets you aggregate many SIDs in the control plane by advertising only the locator prefix, which helps scalability in large deployments.[web:52][web:55]

---

## 4. What is SRv6 Network Programming?

**Simple answer**  
Network programming means **a SID is a small program**, not just an address. When a packet hits a SID, the node executes a pre‑defined behavior, then moves on to the next segment.

**Deep dive**  
RFC 8986 defines a library of **behaviors** such as End, End.X, End.DX4, End.DX6, End.DT46, and others.[web:49][web:52] Each behavior corresponds to a network function: basic forwarding, adjacency steering, L3VPN termination, EVPN L2 service, and so on. This lets you build services (VPNs, service chains, slices) by chaining functions in the segment list, without extra overlay protocols or per‑flow state in the core.[web:49][web:52]

---

## 5. What is the SRH (Segment Routing Header) and how is it used?

**Simple answer**  
The SRH is an IPv6 **Routing Header Type 4** that holds the list of SIDs. It tracks which segment is active and how many remain.

**Deep dive**  
The SRH includes fields such as **Segments Left**, **Last Entry**, flags, and a list of SIDs ordered from last to first.[web:52] The IPv6 destination address always contains the **current active SID**, while the SRH keeps the full list. At each hop that recognizes its SID, the router executes the SID’s behavior, decrements Segments Left, updates the IPv6 destination to the next SID, and forwards the packet accordingly.[web:55]

---

## 6. What are common SRv6 behaviors (End, End.X, End.DX4, End.DX6, End.DT46)?

**Simple answer**  
SRv6 defines a set of standard **behaviors** that tell the router how to treat the packet. Behaviors like End, End.X, End.DX4, End.DX6, and End.DT46 cover basic forwarding, adjacency steering, and VPN services.

**Deep dive**  

- **End**: Basic endpoint behavior; the node acts as a normal IPv6 destination and continues with the next header or next SID.[web:49]  
- **End.X**: Adjacency behavior; forces forwarding over a specific interface/neighbor, enabling strict traffic engineering.  
- **End.DX4 / End.DX6**: L3VPN endpoints; perform IPv4 or IPv6 VRF lookup and forward into the appropriate VPN, replacing MPLS VPN labels.[web:49][web:50]  
- **End.DT46**: Dual‑stack VPN endpoint for both IPv4 and IPv6 in the same service; commonly used in SP deployments where one SID terminates a combined L3VPN service.[web:50]

These behaviors allow SRv6 to natively implement classical MPLS services (L3VPN, EVPN) and more advanced service chains.[web:50][web:51]

---

## 7. What is uSID (micro‑SID) and why do we need it?

**Simple answer**  
uSID is a **compressed SID format** that packs multiple smaller “micro‑SIDs” into a single IPv6 address. It reduces header overhead so we can carry more segments without bloating the packet.

**Deep dive**  
Standard SRv6 uses full 128‑bit SIDs for each segment; with many segments, the SRH becomes large. uSID divides the locator and uses the remaining bits as a stack of fixed‑size micro‑identifiers (for example, 16‑bit or 32‑bit chunks).[web:48][web:54] A single IPv6 destination can carry several uSIDs, and each node shifts and interprets its own micro‑SID in hardware. This approach gives **SRv6 functionality with MPLS‑like compactness**, which is critical for high‑scale cores and AI/5G backbones.[web:48][web:54]

---

## 8. How does SRv6 Traffic Engineering (TE) work?

**Simple answer**  
SRv6 TE works by **encoding a computed path as a list of SIDs** in the SRH, instead of relying only on shortest‑path IGP routing. The ingress node selects the path; the core just executes the segment list.

**Deep dive**  
A controller (PCE or SR‑aware SDN controller) learns topology via BGP‑LS, computes paths based on constraints (latency, bandwidth, affinity), then programs **SRv6 Policies** with explicit segment lists to the headend routers.[web:48][web:50] Those policies use node SIDs, End.X SIDs, or uSIDs to force the packet through specific routers or links. You get RSVP‑TE‑like control without per‑LSP state in the core and with simpler signaling.[web:48][web:53]

---

## 9. How does SRv6 support L3VPN and EVPN services?

**Simple answer**  
SRv6 can carry **L3VPN and EVPN services** using dedicated behaviors like End.DX4, End.DX6, and End.DT46. These replace MPLS service labels with SIDs.

**Deep dive**  
For **L3VPN**, BGP carries VPN routes along with SRv6 service SIDs (for example, End.DT46 or End.DX4/End.DX6).[web:50][web:52] The ingress PE pushes an SRH pointing to the appropriate service SID on the egress PE. When the egress PE processes that SID, it performs the VRF lookup and forwards to the CE. For **EVPN L2 services**, behaviors like End.DX2 or End.DX2V are used to deliver frames into specific virtual circuits or EVPN segments, enabling VPWS and multipoint services over an SRv6 core.[web:49][web:51]

---

## 10. What is SRv6 Service Chaining?

**Simple answer**  
Service chaining with SRv6 means **driving traffic through a sequence of service functions** (firewalls, IDS, load balancers) by encoding their SIDs in the segment list.

**Deep dive**  
Each service node or function is assigned an SRv6 SID representing “send traffic to this service and then continue.”[web:48][web:52] A headend router or controller builds a segment list like `[FW‑SID, IDS‑SID, LB‑SID, VPN‑SID]`. As the packet traverses the network, each service executes its function and forwards based on the remaining SID list. This is aligned with IETF’s service function chaining model and avoids separate NSH or MPLS labels in many designs.[web:52]

---

## 11. What is SRv6 Interworking with SR‑MPLS or MPLS cores?

**Simple answer**  
SRv6 interworking lets an SRv6 domain talk to a traditional MPLS or SR‑MPLS domain. A gateway translates between **SRv6 SIDs** and **MPLS labels** so you can migrate gradually.

**Deep dive**  
Interworking nodes sit at the boundary and map SRv6 service or transport SIDs to equivalent MPLS labels (and vice versa).[web:51] For example, an SRv6 L3VPN SID can map to an MPLS VPN label toward a legacy core, while an MPLS LSP can terminate into a PE that then continues in SRv6 toward another domain. This supports brownfield deployments where part of the network runs MPLS and newer regions run SRv6, especially in large service providers.[web:51][web:52]

---

## 12. What are the main SRv6 deployment use cases?

**Simple answer**  
SRv6 is used heavily in **5G transport**, **cloud/DC networks**, and **AI backbones**, where IPv6‑only, programmability, and service chaining matter.

**Deep dive**  
In **5G**, SRv6 provides transport for network slicing, mobile backhaul, and UPF anchoring with flexible path steering.[web:50][web:53] In **cloud and data centers**, SRv6 EVPN and service chaining support multi‑tenant connectivity and security service insertion at scale. For **AI and HPC fabrics**, SRv6 with uSID and TE enables deterministic, low‑latency paths between GPU clusters, plus fast failure recovery using TI‑LFA.[web:48][web:53][web:56]

---

## 13. What are the main advantages of SRv6 over traditional MPLS?

**Simple answer**  
SRv6 removes the need for MPLS and its supporting protocols, runs natively on IPv6, and enables richer service and telemetry capabilities.

**Deep dive**  
SRv6 can eliminate protocols such as **LDP, RSVP‑TE, and BGP‑LU**, simplifying the control plane.[web:48] It uses only IPv6 plus SRH for forwarding, which fits well with IPv6‑only, cloud, and 5G strategies. Because SIDs encode functions, SRv6 can natively implement VPNs, service chains, and slices in a single data plane, and it leverages IPv6 features like flow labels for better load balancing and telemetry integration.[web:48][web:52][web:53]

---

## 14. What are the main challenges and trade‑offs with SRv6?

**Simple answer**  
The key challenges are larger headers, new hardware requirements, and operational complexity compared to familiar MPLS designs.

**Deep dive**  
Full 128‑bit SIDs make SRv6 headers heavy if many segments are used; uSID mitigates this but requires capable silicon.[web:48][web:54] Some legacy routers cannot parse SRH efficiently, so mixed networks need careful planning. Engineers must also learn the behavior model (End, End.X, End.DT46, etc.) and redesign tools and telemetry around IPv6 SR instead of MPLS. As a result, SRv6 is most attractive where you are already standardizing on IPv6 and refreshing hardware.[web:48][web:55]

---

## 15. When would you choose SRv6 vs SR‑MPLS?

**Simple answer**  
Use **SR‑MPLS** when you have a strong MPLS installed base and mostly IPv4/dual‑stack. Use **SRv6** when you are building or evolving toward **IPv6‑only, cloud, 5G, or AI fabrics**.

**Deep dive**  
SR‑MPLS fits brownfield MPLS networks because it reuses existing label‑switching hardware and operational practices. SRv6 shines in **new IPv6 networks** or strategic migrations where operators want a single IPv6 data plane, tight integration with 5G or cloud stacks, and advanced service chaining and network programming.[web:50][web:52][web:53] Many large operators deploy both, using SR‑MPLS in the legacy core and SRv6 in new regions, with interworking at the boundaries.[web:51]


## 1. Core problem SRv6 had

Plain SRv6 uses full 128‑bit IPv6 addresses as SIDs. For a policy with many segments, the SRH becomes large, so:

- Header overhead is big → more bytes per packet, more MTU pressure, more fragmentation risk.
    
- Routers must process many 128‑bit entries → more work in hardware and slower lookups vs a compact label stack.[](https://netgroup.github.io/srv6-usid-linux-kernel/)
    
- For long SR‑TE paths (10–20+ hops) the overhead is noticeably larger than SR‑MPLS.[](https://srv6.md/rfcs/rfc9800/)
    

## 2. What SID compression / uSID does

SID compression (uSID being one concrete mechanism) tackles exactly that:

- It removes repeated/common prefix bits from the SID list and encodes only the varying, “interesting” parts.
    
- With uSID, multiple 16‑bit “micro‑SIDs” are packed into one 128‑bit container (uSID carrier) instead of using one full IPv6 address per SID.
    
- Up to 6 micro‑SIDs can fit in a single carrier, so for many paths there is no SRH at all or a very small one.
---

# SRv6 Memory Map (Concept Graph for Revision)

Think of SRv6 concepts as a **graph of related ideas**:

- **SRv6 Core Idea**
  - Segment Routing using IPv6 + SRH instead of MPLS

- **SID (Locator / Function / Arguments)**
  - Locator
    - IPv6 prefix, routable
    - Aggregation and summarization
  - Function
    - Encodes behavior (End, End.X, End.DX4, End.DT46…)
  - Arguments
    - VRF ID, service ID, slice ID, etc.

- **Behaviors (Network Programming)**
  - End: basic endpoint
  - End.X: adjacency / TE link
  - End.DX4 / End.DX6 / End.DT46: L3VPN
  - End.DX2 / End.DX2V: EVPN L2 / flexible cross‑connect
  - Used for:
    - L3VPN, EVPN, service chaining, slicing

- **SRH (Segment Routing Header)**
  - Contains segment list (SIDs)
  - Fields: Segments Left, Last Entry, Flags, Tag
  - Drives hop‑by‑hop execution

- **uSID (Micro‑SIDs)**
  - Pack multiple 16/32‑bit micro‑segments in one IPv6 address
  - Reduces overhead, improves scalability

- **Services on SRv6**
  - L3VPN: End.DX4, End.DX6, End.DT46
  - EVPN L2/L3: End.DX2, End.DX2V
  - Service chaining: sequence of service SIDs
  - 5G slicing and mobile transport

- **Control & TE**
  - SRv6 Policies: headend + segment list
  - PCE / SDN Controller: computes paths, uses BGP‑LS
  - TI‑LFA: FRR with SRv6 segments
  - Telemetry: flow label, in‑band OAM

- **Interworking**
  - SRv6 ↔ SR‑MPLS / MPLS cores
  - Gateway nodes map SIDs ↔ labels
  - Supports gradual migration

- **Pros vs Cons**
  - Pros: IPv6‑native, removes MPLS stack, programmable services, fits 5G/AI/cloud
  - Cons: Header size, hardware requirements, new skillset needed

Use this graph mentally as a **mind map**: start from “SRv6 Core Idea”, then branch into **SID structure**, **behaviors**, **services**, **TE/control**, and **interworking**. Every interview question will touch one or more of these branches.

---
```

If you want, next step I can create a **second Markdown note** just with the **memory map as bullet hierarchy** or a **Mermaid mind-map** that Obsidian can render.
<span style="display:none">[^1][^10][^2][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://www.juniper.net/documentation/en_US/day-one-books/DayOne-Intro-SRv6.pdf

[^2]: https://arrcus.com/blog/srv6-and-usid-for-scalable-network-transformation

[^3]: https://datatracker.ietf.org/doc/draft-filsfils-spring-srv6-network-programming/05/

[^4]: https://www.slideshare.net/slideshow/srv6-deployment-usecases-by-aditya-kaul/269534021

[^5]: https://www.cisco.com/c/en/us/td/docs/iosxr/cisco8000/srv6/b-srv6-configuration-guide/srv6-based-layer-2-and-integrated-vpn-services.html

[^6]: https://www.scribd.com/document/747228312/SRv6-a-Short-Introduction

[^7]: https://blogs.cisco.com/sp/srv6-from-5g-networks-to-ai-infrastructure-a-journey-of-innovation

[^8]: https://www.segment-routing.net/images/20221207-srv6-menog22-mounir.pdf

[^9]: https://www.ciscopress.com/articles/article.asp?p=3203556\&seqNum=2

[^10]: https://www.cisco.com/site/us/en/solutions/cisco-on-cisco/ai-ready-infrastructure.html

