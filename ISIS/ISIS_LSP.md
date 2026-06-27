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
Basically contain IP prefix , mask ,metric and adjacent neighbors

LSPs Acknowledge by either by  CSNP and PSNP

LSP max lifetime  Default : 20 min 
```
command "lsp-max-lifetime"
```

LSP refresh interval    Default : 15 min 
```
lsp-refresh-interval"
```

What is zero age lifetime of LSP  : 60 Sec  can not changed of modified

	If a router does not refresh its LSP after the refresh interval and the LSP ages    on to zero remaining lifetime.   after that grace period of 60sec is given and then LSP is purged


)))))))))))))))))))))))))))))))))))))))=================================


# IS-IS LSP (Link State PDU) – Architect Notes + Interview  

  

## 1. LSP (What It Is – Simple Understanding)  

  

- What: LSP (Link State PDU) is the packet that carries network topology information in IS-IS.  

- In simple terms:  

It contains → **IP prefix + mask + metric + neighbor information**  

- Why: Required by all routers to build LSDB and run SPF (shortest path calculation).  

- How:  

- Each router generates its own LSP  

- Floods it across IS-IS domain  

- All routers build identical LSDB  

- Risk:  

- Missing or stale LSP → incorrect routing decisions  

- Example:  

- Router fails to advertise updated prefix → traffic blackhole  

- Takeaway:  

LSP is the **core building block of IS-IS routing decisions**  

  

👉 Interview Angle:  

LSP contains topology details like prefixes, metrics, and neighbors, which routers use to compute shortest paths.  

  

---  

  

## 2. What Exactly an LSP Contains (Breakdown)  

  

Typical LSP includes:  

  

- Router/System ID (who generated it)  

- IP prefixes (reachable networks)  

- Subnet mask  

- Metric (cost)  

- Adjacent neighbors (links)  

- Optional TLVs (TE, SR, etc.)  

  

✅ Key Idea:  

- LSP = **Complete view of router’s connectivity**  

  

---  

  

---  

  

- Final Line:  

In IS-IS, accurate and synchronized LSP distribution is essential for stable and predictable routing.  

  

---  

  

## 5. Quick Recall (Interview)  

  

- LSP = topology information packet  

- Contains:  

- Prefix  

- Mask  

- Metric  

- Neighbors  

- Used for:  

- LSDB  

- SPF calculation  

  

---  

  

## 6. Final Architect Insight  

  

LSP is not just a packet—it represents the **network topology in distributed form**.  

Every design decision in IS-IS (scaling, summarization, flooding control) ultimately impacts how efficiently LSPs are generated, propagated, and processed.