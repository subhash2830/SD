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
# IS-IS Design Notes (Compact – Architect Level)  

  

## 1. ATT BIT  

**What:** Flag set by L1-2 router indicating reachability to L2 core. [1](https://nokia-my.sharepoint.com/personal/subhash_chaitram_dundale_nokia_com/Documents/Git/ISIS/ISIS_Default_route_propagation_rule.md)  

**Why:** L1 routers cannot see external routes → need default path.  

**How:** L1 installs default route → nearest L1-2 router.  

**Risk:** Chooses closest, not best exit (suboptimal routing).  

**Example:** Branch prefers low-bandwidth DC because it is closer.  

**Takeaway:** Simple but no traffic engineering control.  

  

