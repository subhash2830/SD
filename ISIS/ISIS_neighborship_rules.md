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
For Level 1  ( intra area routes available )

   Routers must be in same area and must have level1 or default level 1-2 enable

For Level 2 ( interarea routes tracking )

  Can be same/different area and must have level2 and level 1-2 enable

NW should be match pt-to-pt or broadcast
Hello timer: no need to macth the value ( no negociation on specified time hello should received  )

R1 = 10;40 and r2 30:120 

R1 expect hello from R2 within 40 sec 
*******************************************************************************


# IS-IS Levels (L1 vs L2 Adjacency & Timers) – 

  

## 1. Level-1 (Intra-Area Routing)  

  

- What: L1 routers exchange routes within the same IS-IS area.  

- Why: Provides local routing scope and limits LSDB size for scalability.  

- How:  

- Both routers must:  

- Be in same Area ID  

- Support L1 or L1-2  

- Risk:  

- Area mismatch → no L1 adjacency  

- Incorrect level configuration → route isolation  

- Example:  

- Two routers in different areas configured only as L1 → no adjacency formed  

- Takeaway:  

L1 adjacency strictly depends on **same area + L1 capability**.  

  

👉 Interview Angle:  

L1 adjacency requires same Area ID and compatible level configuration (L1 or L1-2).  

  

---  

  

## 2. Level-2 (Inter-Area / Backbone Routing)  

  

- What: L2 routers exchange routes across areas (backbone function).  

- Why: Enables inter-area communication using a flat backbone design.  

- How:  

- Routers can be in same or different areas  

- Must support L2 functionality (L2 or L1-2)  

- Risk:  

- Missing L2 capability → no inter-area connectivity  

- Example:  

- L1-only router cannot reach backbone → relies on L1-2 router  

- Takeaway:  

L2 removes strict area boundary requirement and acts as IS-IS backbone.  

  

👉 Interview Angle:  

L2 adjacency does not depend on area match, only on L2 capability.  

  

---  

  

## 3. Network Type Requirement  

  

- What: Interface type defines adjacency formation (Point-to-Point or Broadcast).  

- Why: Both sides must follow same adjacency logic.  

- How:  

- Types must match on both routers  

- Risk:  

- Mismatch → adjacency will not form  

- Example:  

- One side P2P, other Broadcast → no neighbor  

- Takeaway:  

Network type must always be consistent for adjacency establishment.  

  

---  

  

## 4. Hello & Hold Timer Behavior  

  

- What:  

- Hello → adjacency maintenance  

- Hold → failure detection threshold  

- Key Behavior:  

- Timers do NOT need to match  

- Each router uses its own hold timer  

  

---  

  

## 5. Why Timer Mismatch Still Works  

  

- What:  

- IS-IS does NOT negotiate timers  

- Why:  

- Each router independently tracks neighbor liveness  

- How:  

Example:  

- R1: Hello = 10s, Hold = 40s  

- R2: Hello = 30s, Hold = 120s  

  

👉 Behavior:  

- R1 expects hello from R2 within **40s**  

- R2 sends every 30s → OK  

  

- Risk:  

- If one router sends slower than other's hold → adjacency flap  

  

- Example Issue:  

- R2 hello = 50s, R1 hold = 40s → R1 drops adjacency  

  

- Takeaway:  

Timer mismatch is allowed, but must satisfy:

Send Interval < Neighbor Hold Time

```

---

## 6. Design Thinking (Critical Insight)

- IS-IS prioritizes:
- Simplicity over strict negotiation
- Engineer must:
- Ensure compatibility manually

👉 Key Rule:
- **Receiver hold timer must always be greater than sender hello interval**

---

## 7. Real-World Scenario (STAR Applied)

- S:
IS-IS adjacency was flapping intermittently between two routers.

- T:
Identify root cause and stabilize adjacency.

- A:
Found hello timer mismatch where one router’s hello interval exceeded neighbor’s hold timer; adjusted timers to maintain safe ratio.

- R:
Adjacency stabilized with consistent neighbor state and improved convergence behavior.

Short Explanation:
Timer mismatch is allowed, but improper ratio leads to adjacency instability.

- Final Line:
In IS-IS, timer compatibility is not negotiated, so correct design must ensure hello intervals always fit within neighbor hold timers.

---

## 8. Quick Recall (Interview)

- L1:
- Same area required
- L1/L1-2 routers only  

- L2:
- Area independent
- Requires L2 capability  

- Network type:
- Must match (P2P/Broadcast)  

- Timer rule:
- No negotiation  
- Hello < Neighbor Hold  

---

## 9. Final Architect Insight

IS-IS adjacency formation depends on **three pillars**:
- Logical compatibility (Area + Level)
- Physical compatibility (Network type)
- Timing compatibility (Hello vs Hold)

Failure in any one leads to adjacency issues, making validation critical in large-scale deployments.
```