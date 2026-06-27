---
uid:
title: ISIS
alias:
topic: Nw types
date:
tags:
status:
priority:

---
 - [ ] point to point
 - [ ] Broad cast 
 - [ ] network type must be match else neighbor ship will not came up 

  

## 1. Network Types (Interface Type)

- What: Defines link behavior and adjacency formation type.
- Types:
- Point-to-Point
- Broadcast
- Why:
- Controls Hello behavior, DIS election, and LSDB representation
- How:
- Must match on both sides for adjacency
- Risk:
- Mismatch → adjacency will NOT form
- Example:
One side P2P, other Broadcast → no neighbor
- Takeaway:
Network type must always match for stable adjacency.

👉 Interview Angle:
Network type defines adjacency formation rules; mismatch prevents neighbor establishment.
