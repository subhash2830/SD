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
 If this bit is set, it indicates an overload condition at the router

 An overload condition indicates the router's performance is degraded by low memory and processing resources

 LSPs with the overload bit set are not reflooded and also are not used in calculating paths through the overloaded router

 Router can used to reach the destination prefix behind router but router can not be used as transit router for destiantion behind other routers

 command to manually set overload bit use process level commmand

```
 cli
  "set-overload-bit"
```

  