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
 NSAP address :   min 8 -- max 20 bytes 
[[ISIS_system_ID]] 

 Area ID - 13 bytes -  1 byte variable(inter-area routing)
 System ID-                6 bytes fixed(intra-area routing)
 NSel ( network Selector)-                         1 byte fixed (always 00 for router == IS)


```
cli
  "net 49.0000.0000.0000.0001.00"
  
```



