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
 - [ ] Hello , LSPs and SNPs 

 - [ ] Types of LSPs is LSP  > type 1 LSP  and type 2 LSP

 - [ ] Hellos  --->  Broadcast hellos L1  L2 , IIHs and point  to point  Hellos 

 - [ ] SNPs -->  CSNPs and PSNPs ( Complete and partial )

====******************************************************=============


# IS-IS Packet Types (Hello, LSP, SNP)  

  

## 1. Packet Types Overview  

  

- What: IS-IS uses three main packet types:  

- Hello (IIH)  

- LSP (Link State PDU)  

- SNP (Sequence Number PDU)  

- Why: Each packet type serves a specific role in neighbor discovery, topology distribution, and database synchronization.  

- How:  

- Hello → form adjacency  

- LSP → carry topology  

- SNP → maintain LSDB consistency  

- Risk: Misbehavior in any type leads to adjacency failure or LSDB inconsistency.  

- Example: Missing SNP processing → stale LSDB → routing issues.  

- Takeaway: IS-IS stability depends on correct interaction of all three packet types.  

  

👉 Interview Angle:  

IS-IS uses Hello for adjacency, LSP for topology, and SNP for database synchronization—together ensuring a consistent LSDB.  

  

---  

  

## 2. Hello Packets (IIH – IS-IS Hello)  

  

- What: Used to discover and maintain neighbor adjacency.  

- Why: Without Hello exchange, routers cannot form IS-IS neighbors.  

- How:  

- Types:  

- Broadcast Hello (L1 / L2)  

- Point-to-Point Hello  

- Also called IIH (IS-IS Hello)  

- Risk:  

- Mismatch (area, MTU, level) → adjacency failure  

- Example:  

- MTU mismatch → Hello works but LSP fails later  

- Takeaway:  

Hello is foundation of adjacency; all routing depends on it.  

  

👉 Interview Angle:  

Hello packets establish adjacency and validate compatibility (area, level, timers).  

  

---  

  

## 3. LSP (Link-State PDU)  

  

- What: Carries topology information (prefix, metric, neighbors).  

- Types:  

- L1 LSP → intra-area  

- L2 LSP → backbone  

- Why: Needed for SPF calculation and routing decisions.  

- How:  

- Flooded across domain  

- Stored in LSDB  

- Risk:  

- Stale or missing LSP → incorrect routing  

- Example:  

- LSP not refreshed → route disappears → traffic drop  

- Takeaway:  

LSP is the core of IS-IS; everything depends on accurate flooding.  

  

👉 Interview Angle:  

LSP contains the network topology and is used by SPF to compute shortest paths.  

  

---  

  

## 4. SNP (Sequence Number PDUs)  

  

- What: Used to synchronize LSDB between routers.  

- Types:  

- CSNP → Complete Sequence Number PDU  

- PSNP → Partial Sequence Number PDU  

- Why:  

- Ensure all routers have identical LSDB  

- How:  

- CSNP:  

- Sent periodically by DIS  

- Acts like LSDB summary  

- PSNP:  

- Sent to request missing LSPs or acknowledge receipt  

- Risk:  

- Missing synchronization → LSDB mismatch → routing inconsistency  

- Example:  

- Router detects missing LSP from CSNP → requests via PSNP  

- Takeaway:  

SNP ensures LSDB consistency across the network.  

  

👉 Interview Angle:  

CSNP advertises full database summary, while PSNP requests or acknowledges specific LSPs.  

  

---  

  

## 5. Design Thinking (How They Work Together)  

  

| Packet | Role | Impact |  

|--------|------|--------|  

| Hello | Adjacency | Network connectivity |  

| LSP | Topology | Path calculation |  

| SNP | Sync | Consistency |  

  

👉 Key Flow:  

1. Hello → neighbor formed  

2. LSP → topology shared  

3. SNP → database synchronized  

  

---  

  

## 6. Real-World Scenario (STAR Applied)  

  

- S:  

Network experienced inconsistent routing due to missing topology updates between routers.  

  

- T:  

Ensure LSDB consistency across all routers.  

  

- A:  

Verified SNP exchange; identified missing LSP synchronization and fixed DIS/CSNP behavior.  

  

- R:  

LSDB synchronized, SPF stabilized, routing consistency restored.  

  

Short Explanation:  

Even if LSPs are generated, without proper SNP synchronization routers may not share identical LSDBs.  

  

- Final Line:  

In IS-IS, stability depends not only on LSP generation but also on proper synchronization using SNPs.  

  

---  

  

## 7. Quick Recall (Interview)  

  

- Hello → adjacency  

- LSP → topology  

- SNP → synchronization  

  

- LSP Types:  

- L1 (intra-area)  

- L2 (backbone)  

  

- Hello Types:  

- Broadcast (L1/L2)  

- Point-to-point  

  

- SNP Types:  

- CSNP → full database summary  

- PSNP → request/ack  

  

---  

  

## 8. Final Architect Insight  

  

IS-IS packet design separates responsibilities clearly—adjacency (Hello), topology (LSP), and synchronization (SNP).  

This modular approach ensures scalability, but failures in any stage can impact convergence, making monitoring and validation critical in large deployments.