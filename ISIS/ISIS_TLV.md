---
uid:
title:ISIS
alias:
topic:
date:
tags:
status:
priority:

---
[[TLV.png#ISIS_TLV]]

| TLV        | Name                 | Description                                                                                                                                                                                                                                                                                                        |
| ---------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1          | Area Address         | Includes the Area Addresses to which the Intermediate System is connected.                                                                                                                                                                                                                                         |
| 2          | IIS Neighbors        | Includes all the IS-ISs running interfaces to which the router is connected.                                                                                                                                                                                                                                       |
| 8          | Padding              | Primarily used in the IS-IS Hello (IIH) packets to detect the maximum transmission unit (MTU) inconsistencies. By default, IIH packets are padded to the fullest MTU of the interface.                                                                                                                             |
| 10         | Authentication       | The information that is used to authenticate the PDU.                                                                                                                                                                                                                                                              |
| 22         | TE IIS Neighbors     | Increases the maximum metric to three bytes (24 bits). Known as the Extended IS Reachability TLV, this TLV addresses a TLV 2 metric limitation. TLV 2 has a maximum metric of 63, but only six out of eight bits are used.                                                                                         |
| 128        | IP Int. Reachability | Provides all the known IP addresses that the given router knows about via one or more internally-originated interfaces. This information may appear multiple times.                                                                                                                                                |
| 129        | Protocols Supported  | Carries the Network Layer Protocol Identifiers (NLPID) for Network Layer protocols that the IS (Intermediate System) is capable. It refers to the Data Protocols that are supported. For example, IPv4 NLPID value 0xCC, CLNS NLPID value 0x81, and/or IPv6 NLPID value 0x8E will be advertised in this NLPID TLV. |
| 130        | IP Ext. Address      | Provides all the known IP addresses that the given router knows about via one or more externally-originated interfaces. This information may appear multiple times.                                                                                                                                                |
| 132        | IP Int. Address      | The IP interface address that is used to reach the next-hop address.                                                                                                                                                                                                                                               |
| 134        | TE Router ID         | This is the Multi-Protocol Label Switching (MPLS) traffic engineering router ID.                                                                                                                                                                                                                                   |
| 135        | TE IP Reachability   | Provides a 32 bit metric and adds a bit for the "up/down" resulting from the route-leaking of L2->L1. Known as the Extended IP Reachability TLV, this TLV addresses the issues with both TLV 128 and TLV 130.                                                                                                      |
| 137        | Dynamic Hostname     | Identifies the symbolic name of the router originating the link-state packet (LSP).                                                                                                                                                                                                                                |
| 10 and 133 |                      | TLV 10 should be used for Authentication; not the TLV 133. If TLV 133 is received, it is ignored on receipt, like any other unknown TLVs. TLV 10 should be accepted for authentication only.                                                                                                                       |


# IS-IS TLVs (Type-Length-Value) – Comprehensive Notes  

    

IS-IS uses **TLVs (Type-Length-Value)** to carry routing information inside PDUs.  

This design makes IS-IS **highly extensible**, allowing new features (IPv6, MPLS, SR) without redesigning the protocol.  

  

---  

  

## 2. Core Concept  

- TLV = Flexible data structure inside IS-IS packets  

- Each TLV contains:  

- Type → identifies information  

- Length → size of value  

- Value → actual data  

  

✅ Key Advantage:  

- New features can be added by defining new TLVs (no protocol rewrite)  

  

---  

  

## 3. Important TLVs Explained  

  

### 🔷 TLV 1 – Area Address  

- Defines area(s) router belongs to  

- Used for L1 adjacency formation  

  

✅ Design Insight:  

- Mismatch → adjacency failure  

  

---  

  

### 🔷 TLV 2 – IS Neighbors  

- Lists directly connected IS-IS neighbors  

  

✅ Limitation:  

- Metric limited (6-bit effectively, max ~63)  

  

---  

  

### 🔷 TLV 8 – Padding  

- Pads Hello packets to full MTU  

  

✅ Why Important:  

- Detect MTU mismatch early  

  

✅ Real Issue:  

- Without padding → adjacency forms but LSP fails  

  

---  

  

### 🔷 TLV 10 – Authentication  

- Used for IS-IS PDU authentication  

  

✅ Best Practice:  

- Always configure authentication (avoid rogue routers)  

  

⚠️ Important Note:  

- TLV 133 is deprecated → ignored  

  

---  

  

### 🔷 TLV 22 – Extended IS Reachability  

- Enhanced version of TLV 2  

- Supports **24-bit metric (large network scaling)**  

  

✅ Why Needed:  

- Overcomes TLV 2 metric limitation  

  

✅ Used in:  

- Traffic Engineering (MPLS, SR)  

  

---  

  

### 🔷 TLV 128 – IP Internal Reachability  

- Advertises internal IPv4 prefixes  

  

✅ Limitation:  

- Old format (less flexible)  

  

---  

  

### 🔷 TLV 130 – IP External Reachability  

- Advertises external prefixes  

  

✅ Use Case:  

- Route redistribution  

  

---  

  

### 🔷 TLV 132 – IP Interface Address  

- Carries interface IP used for next-hop  

  

✅ Design Role:  

- Helps resolve forwarding path  

  

---  

  

### 🔷 TLV 129 – Protocols Supported  

- Indicates supported protocols (IPv4, IPv6, CLNS)  

  

✅ Example:  

- IPv4 → 0xCC  

- IPv6 → 0x8E  

  

✅ Importance:  

- Ensures compatibility between routers  

  

---  

  

### 🔷 TLV 134 – TE Router ID  

- Router ID used for MPLS TE  

  

✅ Use Case:  

- Traffic Engineering tunnels  

  

---  

  

### 🔷 TLV 135 – Extended IP Reachability  

- Modern replacement of TLV 128/130  

- Features:  

- 32-bit metric  

- Up/Down bit (route leaking control)  

  

✅ Design Advantage:  

- Better scalability and policy control  

  

---  

  

### 🔷 TLV 137 – Dynamic Hostname  

- Advertises router hostname  

  

✅ Operational Benefit:  

- Easier troubleshooting (human-readable)  

  

---  

  

## 4. Design & Architecture Insights  

  

### ✅ Why TLVs Matter  

- Enable:  

- MPLS  

- Segment Routing  

- IPv6  

- Without TLVs → protocol redesign needed  

  

---  

  

### ✅ Migration Insight  

- Old TLVs (128, 130) → replaced by 135  

- Modern networks rely on:  

- Extended TLVs (22, 135)  

  

---  

  

## 5. Risks & Challenges  

  

| Risk | Impact |  

|------|-------|  

| MTU mismatch (TLV 8) | LSP drops, instability |  

| Weak authentication | Security breach |  

| Old TLVs used | Limited scalability |  

| Misconfigured TLV | Adjacency failure |  

  

---  

  
# IS-IS Traffic Engineering (TE Extensions)  

  

## 1. Architect Notes (Clear + Practical)  

  

- What: TE extensions add additional link attributes (bandwidth, delay, resources) into IS-IS LSDB.  

- Why: Normal IS-IS uses only cost metric → insufficient for advanced traffic engineering.  

- How:  

- Uses extended TLVs (like TLV 22, 135)  

- Advertises link bandwidth and constraints  

- Risk:  

- Increased LSDB size  

- Higher complexity in design and troubleshooting  

- Example:  

- MPLS TE uses bandwidth info to compute optimal tunnel path  

- Takeaway:  

TE extensions enable intelligent path selection beyond simple metrics.  

  

👉 Interview Angle:  

TE extensions allow IS-IS to advertise link attributes for MPLS and advanced traffic engineering.  

  

---  

  

## 2. Interview Answer  

  

- IS-IS TE extensions enhance routing by including bandwidth and link constraints.  

- Used in MPLS networks for optimized traffic path computation.  

  

- S: Network required bandwidth-aware path selection.  

- T: Avoid congestion on links.  

- A: Enabled IS-IS TE extensions.  

- R: Traffic distributed based on link capacity.  

  

- Final Line:  

TE extensions transform IS-IS from simple routing to intelligent path computation.
  

---  

  



  

---  

  

