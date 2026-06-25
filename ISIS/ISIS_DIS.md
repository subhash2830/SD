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
## What is the DIS and why does ISIS need it?

- On a broadcast LAN (shared segment), many routers could form a full mesh of adjacencies.
    
- If every router advertised every other router directly, the LSDB and SPF load would grow quickly.
    
- ISIS solves this by electing a **Designated Intermediate System (DIS)** on each LAN (per level):
    
    - The DIS represents the LAN as a single **pseudonode**.
        
    - All routers on that LAN form an adjacency to the pseudonode instead of to every other router.
        
    - This keeps the topology simpler and SPF more scalable.
        

---

## DIS election rules

1. **Highest DIS priority wins**
    
    - Priority range: **0–127**, default typically **64**.
        
    - Interface CLI example:
        
        text
        
        `isis priority 100`
        
    - Higher priority → more likely to be DIS on that LAN.
        
2. **If priorities tie → highest MAC address wins**
    
    - Uses the interface MAC on that segment as tiebreaker.[](https://www.menog.org/presentations/menog-4/MENOG4-ISIS-Tutorial.pdf)
        
3. **If still tied → highest System ID may be final tiebreaker**
    
    - This is vendor‑specific but common in practice.
        

Key behavior vs OSPF:

- There is **no “priority 0”** equivalent to permanently exclude a router; any router with the interface up is eligible.
    
- The DIS role is **preemptive**: a router with higher priority that appears later can _take over_ the DIS role.[](https://www.menog.org/presentations/menog-4/MENOG4-ISIS-Tutorial.pdf)
    

**Design (CCDE) reason:**

- You deliberately tune priorities so that the **most capable router** (best CPU/control plane, most stable) becomes DIS on high‑fanout LANs, instead of relying on defaults.
    

---

## No backup DIS

- ISIS has **only one DIS** on a LAN (per level) and **no backup DIS**.
    
- If the DIS fails, a new DIS is elected using the same rules.
    
- During this change, neighbor adjacencies and LSDB state reconverge, but hello timers and CSNPs help minimize disruption.[](https://www.menog.org/presentations/menog-4/MENOG4-ISIS-Tutorial.pdf)
    

**Design implication:**

- High availability is achieved by:
    
    - Having multiple capable routers on the LAN.
        
    - Tuning timers/BFD for fast failure detection.
        
    - Setting priorities so that failover goes to a sensible backup router, not a weak box.
        

---

## DIS role in LSPs and CSNPs

On a broadcast segment:

- Every router sends its own **LSPs** onto the LAN.
    
- The DIS:
    
    - **Creates and maintains a pseudonode LSP** that represents the LAN itself.
        
    - Describes links from the pseudonode to each router on that segment.
        
- Each real router therefore has:
    
    - Its **own LSP** (describing its links).
        
    - A **link to the pseudonode** in that LSP.
        

To keep LSDBs synchronized, the DIS periodically sends **CSNPs (Complete Sequence Number PDUs)**:

- A CSNP is effectively an index of all LSPs that exist on that LAN.
    
- If a router sees that its own LSP is _not_ listed in the CSNP, it **refloods its LSP** to the LAN to repair the database.
    

**Design (CCDE) reason:**

- DIS + pseudonode + CSNPs give a **scalable representation** of a shared LAN and robust LSDB synchronization.
    
- This avoids N² adjacencies and keeps SPF computation efficient, which is crucial on large campus or metro LAN segments.
    

---

## Pseudonode concept

- A **pseudonode** is a virtual router representing the broadcast LAN itself.
    
- The elected DIS is responsible for:
    
    - Creating the pseudonode LSP.
        
    - Updating it when routers join/leave the LAN.
        
- Other routers never become the pseudonode; they simply see:
    
    - One “real” LSP from the DIS.
        
    - One “pseudonode” LSP that describes the LAN.
        

**Architectural value:**

- Modeling the LAN as a single pseudonode simplifies **topology, SPF, and SR/SRv6 TLVs** on that segment.
    
- It keeps ISIS scalable even when many routers share a common broadcast network.
    

---

Given this markdown version, how would you summarize in your own words why **tuning DIS priority** is important on large shared VLANs?