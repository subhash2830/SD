---
uid:
title:
alias:
topic:
date:
tags:
status:
priority:

---
 NSAP address :   min 8 -- max 20 bytes 
[[ISIS_system_ID]] 

 Area ID - 13 bytes -  1 byte variable(inter-area routing)
 System ID-                6 bytes fixed(intra-area routing)
 NSel ( network Selector)-                         1 byte fixed (always 00 for router == IS)


```
cli
  "net 49.0000.0000.0000.0001.00"
  
```

***************************************************************
## 1. NSAP Address (NET – Network Entity Title)  

  

- What: NSAP (NET) is the Layer-3 address used by IS-IS; identifies router uniquely in the domain.  

- Why: IS-IS does not use IP for routing identity; it relies on NSAP for topology and LSP origin.  

- How:  

- Length: 8 to 20 bytes  

- Structure:  

- Area ID → variable (up to ~13 bytes)  

- System ID → 6 bytes (fixed, router identity)  

- NSEL → 1 byte (always 00 for routers)  

- Example:

net 49.0000.0000.0000.0001.00

```
- Risk:
- Duplicate System ID → LSP overwrite → routing instability
- Incorrect Area ID → adjacency failure (especially L1)
- Takeaway:
NSAP defines both hierarchy (Area ID) and identity (System ID); design must ensure uniqueness and consistency.

👉 Interview Angle:
NSAP is the fundamental identifier in IS-IS combining area and router identity, unlike OSPF which separates Router-ID and IP.

---

## 2. System ID (Critical Component)

- What: 6-byte fixed portion of NSAP; uniquely identifies router in IS-IS domain.
- Why: Used as LSP origin identifier; must be globally unique (not per area).
- How:
- Derived manually or from IP/MAC
- Risk:
- Duplicate System IDs → LSDB corruption and routing instability
- Example:
Two routers share same System ID → one LSP overwrites another
- Takeaway:
System ID = global identity; must be carefully planned.

---

## 3. Area ID Role

- What: Defines IS-IS area membership.
- Why:
- Required for L1 adjacency formation
- Supports hierarchical design
- How:
- Routers must share Area ID to form L1 adjacency
- Risk:
- Mismatch → no adjacency
- Example:
Two routers in different Area IDs → L1 adjacency fails
- Takeaway:
Area ID enforces logical boundaries in IS-IS.

---

## 4. NSEL (N-Selector)

- What: Last byte of NSAP; identifies upper-layer service.
- Why:
- Distinguishes system vs application (protocol usage)
- How:
- For IS-IS routers → always set to `00`
- Risk:
- Incorrect value may cause protocol misinterpretation
- Example:
Non-zero NSEL → not treated as IS (router)
- Takeaway:
Always use `.00` for router NET.

---



---

## 6. Design Thinking (Key Insights)

- NSAP = Identity + Hierarchy
- System ID = Global uniqueness requirement
- Area ID = Local grouping for L1
- Network type = Adjacency control mechanism

---

## 7. Real-World Scenario (STAR Applied)

- S:
IS-IS adjacency was not forming between two routers in production network.

- T:
Identify root cause and restore connectivity.

- A:
Verified configuration and found mismatch in network type (P2P vs broadcast) and incorrect Area ID.

- R:
After aligning network type and NSAP configuration, adjacency formed successfully and routing stabilized.

Short Explanation:
Both NSAP parameters and network type must match for adjacency; mismatch breaks connectivity.

- Final Line:
IS-IS adjacency depends on both logical identity (NSAP) and physical behavior (network type), making consistency critical for stable operation.

---

## 8. Quick Recall (Interview)

- NSAP = 8–20 bytes  
- System ID = 6 bytes (unique globally)  
- Area ID = variable (L1 grouping)  
- NSEL = 00 (for routers)  
- Network types:
- P2P
- Broadcast  
- Mismatch → no adjacency  

---

## 9. Final Architect Insight

In IS-IS, NSAP design is more than just addressing—it defines routing hierarchy, identity, and adjacency behavior.  
A well-planned NSAP structure combined with correct network type configuration ensures deterministic and scalable network design.
```


