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
 as a link-state routing protocol, there are some limitations.

 You can only configure summarization between areas or on a router that is doing redistribution.

 Routes are seens as "i su" in routing table

=========================================================

# IS-IS Summarization & Route Representation   

  

## 1. Summarization in IS-IS (Limitation)  

  

- What: IS-IS supports route summarization only at specific points (L1/L2 boundary or redistribution points).  

- Why: As a pure link-state protocol, IS-IS distributes full topology information; summarization would break topology consistency if done arbitrarily.  

- How:  

- Summarization done at:  

- L1 → L2 boundary (area exit)  

- During route redistribution (external routes)  

- Risk:  

- Poor summarization design → suboptimal routing or blackholes  

- Over-summarization → loss of path visibility  

- Example:  

- Multiple /24 prefixes summarized into /16 at L1/L2 boundary → traffic may go to wrong exit if internal paths differ  

- Takeaway:  

IS-IS limits summarization to preserve accurate topology; summarization must be done carefully at design boundaries.  

  

👉 Interview Angle:  

Unlike distance-vector protocols, IS-IS cannot summarize anywhere because it relies on full topology; summarization is only safe at hierarchy boundaries.  

  

---  

  

## 2. Why Summarization is Limited (Design Reasoning)  

  

- Problem:  

- Link-state protocols require consistent LSDB across routers  

- If summarization done randomly:  

- Topology mismatch occurs  

- SPF results become inconsistent  

  

👉 Key Insight:  

- IS-IS prioritizes **topology accuracy over route aggregation**  

  

---  

  

## 3. Route Representation in Routing Table (IS-IS Codes)  

  

- What:  

- IS-IS routes appear with specific codes in routing table:  

- **i L1** → Level 1 route  

- **i L2** → Level 2 route  

- **i ia / i su (vendor variation)** → inter-area / summary routes  

- Why:  

- Helps identify route origin and hierarchy level  

- How:  

- Derived from LSP information and route leaking behavior  

- Risk:  

- Misinterpretation of route type → wrong troubleshooting decisions  

- Example:  

- Engineer assumes L1 route is external, but it is actually internal → wrong root cause analysis  

- Takeaway:  

Understanding route codes is essential for troubleshooting and path selection analysis.  

  

👉 Interview Angle:  

IS-IS route types indicate hierarchy and origin, helping identify whether routes are intra-area, inter-area, or summarized.  

  

---  

  

## 4. Design Considerations  

  

- Summarization should be:  

- Done only at:  

- L1/L2 boundaries  

- Redistribution points  

- Avoid:  

- Deep summarization in complex topologies  

- Validate:  

- Traffic paths after summarization  

- Combine with:  

- Proper metric tuning  

  

---  

  

## 5. STAR Interview Answer (Applied Scenario)  

  

- S:  

Large enterprise network experienced inefficient routing and traffic blackholing after route summarization.  

  

- T:  

Ensure summarization does not impact routing accuracy and path selection.  

  

- A:  

Reviewed IS-IS design and restricted summarization to L1/L2 boundary; validated route reachability and metrics.  

  

- R:  

Restored optimal traffic flow, eliminated blackholes, and improved routing predictability.  

  

Short Explanation:  

IS-IS summarization must be done only at defined hierarchy points; otherwise, it breaks topology consistency.  

  

- Final Line:  

In IS-IS, summarization is controlled and limited by design to preserve LSDB accuracy, so improper aggregation can directly impact routing correctness.  

  

---  

  

## 6. Quick Recall (Interview)  

  

- IS-IS = link-state → full topology required  

- Summarization:  

- Only at L1/L2 boundary or redistribution  

- Overuse → blackholes / suboptimal routing  

- Route codes:  

- i L1, i L2, i su/inter-area  

  

---  

  

## 7. Final Architect Insight  

  

IS-IS deliberately restricts summarization to maintain a consistent and accurate topology view across the network.  

This ensures stable SPF calculations but requires careful design when aggregation is introduced at hierarchy boundaries.