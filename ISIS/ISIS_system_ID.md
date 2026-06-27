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
it is same as OSPF Router-ID

 L2 LSP forwarded as it is in another area

 so atleast L2 and L1/2 routers system ID must be unique regard less of area


===========================================================
# IS-IS System ID & L2 LSP Behavior (Architect Notes + Interview)  

  

## 1. System ID in IS-IS  

  

- What: System ID is a unique identifier (typically 6 bytes) used to identify an IS-IS router; similar in purpose to OSPF Router-ID.  

- Why: Required to uniquely identify routers in LSDB and avoid ambiguity during SPF calculation and LSP flooding.  

- How: Part of NSAP address; combined with N-selector to form NET; used in LSP origin and adjacency formation.  

- Risk: Duplicate System ID → LSP overwrite, routing instability, unpredictable SPF results.  

- Example: Two routers with same System ID → one router’s LSP overwrites another → traffic misrouting.  

- Takeaway: System ID must be globally unique within IS-IS domain, not just within area.  

  

👉 Interview Angle:  

System ID in IS-IS acts like Router-ID in OSPF, but it is embedded in NSAP and must be unique across the entire routing domain.  

  

---  

  

## 2. L2 LSP Flooding Behavior  

  

- What: L2 LSPs are flooded unchanged across the entire Level-2 domain (backbone).  

- Why: L2 acts as backbone; ensures consistent topology view for inter-area routing.  

- How: L2 routers flood LSPs without modification across areas (no area boundary filtering like OSPF).  

- Risk: Large-scale L2 domains → bigger LSDB, higher SPF load.  

- Example: Multi-area network where L2 backbone carries all routes → scaling challenge if not designed properly.  

- Takeaway: L2 domain behaves as a flat backbone; design must control its size.  

  

---  

  

## 3. Why System ID Must Be Globally Unique  

  

- What: L2 LSPs are not changed when moving between areas.  

- Why: Since LSP identity is based on System ID, duplication causes conflicts network-wide.  

- How:  

- L2 routers carry LSPs across areas  

- Duplicate System IDs cause LSP collision (same origin identifier)  

- Risk:  

- LSP overwriting  

- Routing loops or blackholes  

- Unstable LSDB  

- Example:  

- Two routers in different areas with same System ID  

- L2 floods LSP → conflict → one LSP dominates → wrong routing  

- Takeaway: Unlike OSPF (per-area uniqueness), IS-IS requires domain-wide uniqueness.  

  

---  

  

## 4. Design Thinking (Critical Insight)  

  

- IS-IS does NOT have strict area isolation like OSPF  

- L2 backbone floods information transparently  

- System ID becomes:  

👉 **Global identity of router**  

  

---  

  

## 5. STAR Interview Answer (Applied Scenario)  

  

- S: Network experienced inconsistent routing and LSDB instability across multiple areas.  

- T: Identify root cause and ensure stable routing behavior.  

- A: Discovered duplicate System IDs across different areas; reconfigured unique System IDs for all routers.  

- R: LSP conflicts resolved, LSDB stabilized, and routing became consistent.  

  

Short Explanation:  

Even if routers are in different areas, duplicate System IDs create LSP conflicts because L2 floods LSPs unchanged across the domain.  

  

- Final Line:  

In IS-IS, System ID must be globally unique because L2 flooding makes the entire domain logically flat for LSP propagation.  

  

---  

  

## 6. Quick Recall (Interview)  

  

- System ID = router identity (like OSPF RID)  

- Must be globally unique (not per area)  

- L2 LSP floods unchanged across areas  

- Duplicate System ID = LSDB corruption risk  

  

---  

  

## 7. Final Architect Insight  

  

Unlike OSPF where Router-ID uniqueness is critical but area boundaries provide some isolation,  

IS-IS relies on a globally consistent identity because L2 flooding removes strict area separation—making  

System ID planning a foundational design requirement.
