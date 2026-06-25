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
 - [ ]  MTU for ISIS on ethernet interface is 1497
-        Cisco ISIS uses 802.3 LLC frame format
         so actual header length is increased by 3 bytes and actual maximum payload is 1497

- [ ] isis MTU need to check while bring up neighbor ship  else stuck in init 

How ISIS detect MTU mismatch

 All hellos are padded to full MTU size (Padding TLV 8)
 and even if hello padding is disabled cisco router still sends first 5 hellos fully padded
