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
 process level command " **IS-type** " tell router which level1/level2/level1-2 databse we can maintained

 interface level command "**isis circuit-type** " tell router level1/2/1-2 hello can be sent on that interface

 we can use circuit-type with is-type but we need to have database for that level to make it work



# IS-IS IS-Type vs Circuit-Type – Architect Notes + Interview  

  

## 1. IS-Type (Process-Level Behavior)  

  

- What: Defines which LSDB (Level-1, Level-2, or both) the router maintains.  

- Why: Controls routing role in hierarchy (intra-area, backbone, or both).  

- How:

router isis is-type level-1 | level-2-only | level-1-2

```
- L1 → only intra-area database  
- L2 → only backbone database  
- L1-2 → both databases  
- Risk:
- Wrong IS-Type → router cannot participate in required routing scope
- L1-only router cannot access backbone directly
- Example:
- Router configured as L1-only in core → cannot route inter-area traffic
- Takeaway:
IS-Type defines **what routing information router is capable of storing and processing**

👉 Interview Angle:
IS-Type controls LSDB scope and determines whether router participates in L1, L2, or both routing domains.

---

## 2. Circuit-Type (Interface-Level Behavior)

- What: Defines which type of Hello packets (L1, L2, or both) are sent on an interface.
- Why: Controls where adjacencies are formed per interface.
- How:
```

interface X isis circuit-type level-1 | level-2-only | level-1-2

```
- Risk:
- Mismatch → adjacency not formed
- Sending L2 Hello on router without L2 capability → ineffective
- Example:
- Interface configured as L2-only on L1 router → no adjacency
- Takeaway:
Circuit-Type controls **adjacency formation behavior on a specific link**

👉 Interview Angle:
Circuit-type determines which level adjacencies are formed on an interface, independent of global router role.

---

## 3. Dependency Between IS-Type and Circuit-Type

- What: Circuit-Type works only if corresponding LSDB exists in router.
- Why:
- Router cannot advertise or process routes for a level it does not support
- How:
- IS-Type must include level before circuit-type can use it
- Example Cases:

### ✅ Valid
- IS-Type: L1-2  
- Circuit-Type: L2 → ✅ Works  

### ❌ Invalid
- IS-Type: L1-only  
- Circuit-Type: L2 → ❌ No adjacency / ineffective  

- Risk:
- Misconfiguration leads to silent adjacency issues
- Takeaway:
Circuit-Type cannot override IS-Type; it is dependent on it

---

## 4. Design Thinking (Critical Insight)

| Component | Role | Scope |
|----------|------|------|
| IS-Type | What router knows | Global (process) |
| Circuit-Type | Where router participates | Interface |

👉 Key Rule:
- IS-Type = Capability  
- Circuit-Type = Usage of that capability  

---

## 5. Real-World Scenario (STAR Applied)

- S:
IS-IS adjacency was not forming on a backbone link despite correct IP connectivity.

- T:
Identify misconfiguration preventing adjacency.

- A:
Found router configured as L1-only, but interface set as L2; corrected IS-Type to L1-2.

- R:
L2 adjacency formed successfully, and inter-area routing restored.

Short Explanation:
Circuit-Type alone cannot enable adjacency if router does not support that level in IS-Type.

- Final Line:
IS-Type defines capability, and circuit-type defines behavior—both must align for correct IS-IS operation.

---

## 6. Design Guidelines (Architect View)

- Always define IS-Type based on network role:
- Access → L1  
- Core → L2 or L1-2  
- Use Circuit-Type for:
- Granular control per interface  
- Validate:
- IS-Type supports all required circuit-types  
- Avoid:
- Mismatched configuration across links  

---

## 7. Quick Recall (Interview)

- IS-Type → LSDB capability  
- Circuit-Type → Hello/adjacency behavior  
- IS-Type must support circuit-type  
- L1-only router cannot form L2 adjacency  
- Common issue → mismatch between both  

---

## 8. Final Architect Insight

IS-Type and Circuit-Type together define the functional behavior of an IS-IS router—one controls the routing capability, and the other controls how that capability is applied across interfaces. Proper alignment is critical for predictable adjacency formation and hierarchical routing design.
```