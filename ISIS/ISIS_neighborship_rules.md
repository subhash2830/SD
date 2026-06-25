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