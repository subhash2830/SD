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