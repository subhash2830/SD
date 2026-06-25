---
uid: 
title: OSPF
aliases: 
topic: Nw types
date: 
tags:
  - protocol/ospf
status: 
priority:
---
| Nw Types                       | Timer   | Hello     | Neighbor | DR/BDR | Mask           | Next Hop  | Defult    |     |     |     |     |     |     |     |
| ------------------------------ | ------- | --------- | -------- | ------ | -------------- | --------- | --------- | --- | --- | --- | --- | --- | --- | --- |
| Broadcast                      | 10/40   | Multicast | no       | Yes    | Corrected Mask | Unchanged | Ethernet  |     |     |     |     |     |     |     |
| Point to point                 | 10/40   | Multicast | no       | No     | Corrected Mask | NA        | Serial FR |     |     |     |     |     |     |     |
| Non Broadcast                  | 30 /120 | unicast   | Yes      | Yes    | Corrected Mask | Unchanged | Serial FR |     |     |     |     |     |     |     |
| pt to Multi point              | 30 /120 | Multicast | no       | No     | /32            | Hub       | Serial FR |     |     |     |     |     |     |     |
| pt to multipoint non broadcast | 30 /120 | unicast   | Yes      | No     | /32            | Hub       | NA        |     |     |     |     |     |     |     |
| loopback                       | NA      | NA        | NA       | NA     | /32            | NA        | NA        |     |     |     |     |     |     |     |
|                                |         |           |          |        |                |           |           |     |     |     |     |     |     |     |
  

