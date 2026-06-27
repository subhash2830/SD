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
## 4. DEFAULT-INFORMATION ORIGINATE  

**What:** Explicit default route injection command. [1](https://nokia-my.sharepoint.com/personal/subhash_chaitram_dundale_nokia_com/Documents/Git/ISIS/ISIS_Default_route_propagation_rule.md)  

**Why:** ATT lacks control → need policy-based routing.  

**How:** Advertises default even without it in RIB.  

**Risk:** Blackhole if upstream fails.  

**Example:** ISP router advertises default after link failure.  

**Takeaway:** Use with route validation.  