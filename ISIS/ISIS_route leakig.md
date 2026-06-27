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

 By default level 1 LSP are leaked into Level2 LSP

 but Level2 LSP are not leaked into Level1 LSP

 but we can control Level1 LSP to Level2 LSP leaking using command
```
 "redistribute isis ip level-1 into level-2 route-map TEST1"
```


 We can leak Level2 LSP into Level1 database use command
```
 "redistribute isis ip level-2 into level-1 route-map TEST1"
```


  after this routing table will show route as "ia"(inter-area)

****************************************************************
*************************************************************************
# IS-IS Route Leaking (L1 ↔ L2) – Architect Notes + Interview  

  

## 1. Default Behavior (L1 vs L2)  

  

- What: IS-IS hierarchy separates routing into Level-1 (intra-area) and Level-2 (backbone).  

- Why: Provides scalability by isolating topology and limiting LSDB size.  

- How:  

- L1 prefixes are **automatically leaked into L2**  

- L2 prefixes are **NOT leaked into L1 by default**  

- Risk:  

- L1 routers only see default route (via ATT) → lack visibility of specific destinations  

- Example:  

- Branch (L1) only knows default → cannot choose optimal DC exit  

- Takeaway:  

Default design favors simplicity (L1 → L2), but restricts L1 visibility.  

  

👉 Interview Angle:  

L1 routes are advertised to L2 for backbone reachability, but L2 routes are hidden from L1 to maintain hierarchy and prevent LSDB explosion.  

  

---  

  

## 2. Why L2 → L1 is Blocked by Default  

  

- What: L2 routes are not automatically advertised into L1.  

- Why:  

- Prevents flooding large backbone routes into access areas  

- Protects L1 routers from large LSDB and SPF overhead  

- How:  

- L1 routers rely on:  

- Default route (ATT bit)  

- Not full route table  

- Risk:  

- Suboptimal routing (no path visibility)  

- Poor traffic engineering options  

- Example:  

- Two exits exist, but L1 picks closest instead of best  

  

- Takeaway:  

IS-IS enforces hierarchy strictly to maintain scalability.  

  

---  

  

## 3. Manual Route Leaking (Controlled Design)  

  

### 🔷 L1 → L2 (Controlled Leak)  

``

redistribute isis ip level-1 into level-2 route-map TEST1

```

- What: Controls which L1 routes are leaked into L2
- Why: Filters unnecessary prefixes and optimizes backbone routing
- Risk: Over-filtering may hide valid routes

---

### 🔷 L2 → L1 (Explicit Leak)
```

redistribute isis ip level-2 into level-1 route-map TEST1

```

- What: Injects selected L2 routes into L1 database
- Why: Provide visibility to L1 routers for better path selection
- How:
  - Controlled using route-map (policy-based leaking)
- Risk:
  - LSDB growth in L1
  - Potential scaling issues
  - Increased SPF processing

---

## 4. Route Table Behavior

- What: Leaked routes appear as:
```

i ia (inter-area)

```
- Why: Indicates route originated from different level (L1 ↔ L2 boundary)
- Use:
- Helps in troubleshooting
- Differentiates route source

---

## 5. Design Trade-Off (Very Important)

| Option | Benefit | Risk |
|-------|--------|------|
| No L2→L1 leaking | Scalable, simple | Limited visibility |
| Controlled leaking | Better routing decisions | Higher LSDB / CPU |

👉 Key Insight:
- More visibility = better routing  
- More routes = more complexity  

---

## 6. Real-World Scenario (STAR Applied)

- S:
L1 branch routers always selected nearest exit, causing congestion on low-capacity WAN links.

- T:
Improve exit path selection without breaking IS-IS hierarchy.

- A:
Configured L2 → L1 route leaking using route-map to advertise specific critical prefixes.

- R:
Branch routers gained visibility to multiple exits and selected optimal paths, improving application performance.

Short Explanation:
Selective route leaking allows L1 routers to make better decisions while maintaining scalability.

- Final Line:
IS-IS route leaking provides a balance between strict hierarchy and operational flexibility when applied with proper control.

---

## 7. Design Guidelines (Architect View)

- Keep default behavior for scalability
- Use L2 → L1 leaking only when needed (critical prefixes)
- Always use route-maps (never leak everything)
- Monitor LSDB size and CPU after leaking
- Combine with metric tuning for traffic engineering

---

## 8. Quick Recall (Interview)

- L1 → L2: Allowed by default  
- L2 → L1: Not allowed (must configure)  
- Command:
- `redistribute isis ip level-2 into level-1`
- Leaked routes show as: `i ia`
- Use case: Improve path selection for L1 routers

---

## 9. Final Architect Insight

IS-IS hierarchy is designed for scalability, but strict separation limits routing visibility.  
Route leaking provides controlled flexibility, allowing better traffic engineering while preserving stability—when applied carefully with policies.
```