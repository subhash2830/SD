---
uid:
title:ISIS
alias:
topic:
date:
tags:
status:
priority:

---
[[TLV.png#ISIS_TLV]]

| TLV        | Name                 | Description                                                                                                                                                                                                                                                                                                        |
| ---------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1          | Area Address         | Includes the Area Addresses to which the Intermediate System is connected.                                                                                                                                                                                                                                         |
| 2          | IIS Neighbors        | Includes all the IS-ISs running interfaces to which the router is connected.                                                                                                                                                                                                                                       |
| 8          | Padding              | Primarily used in the IS-IS Hello (IIH) packets to detect the maximum transmission unit (MTU) inconsistencies. By default, IIH packets are padded to the fullest MTU of the interface.                                                                                                                             |
| 10         | Authentication       | The information that is used to authenticate the PDU.                                                                                                                                                                                                                                                              |
| 22         | TE IIS Neighbors     | Increases the maximum metric to three bytes (24 bits). Known as the Extended IS Reachability TLV, this TLV addresses a TLV 2 metric limitation. TLV 2 has a maximum metric of 63, but only six out of eight bits are used.                                                                                         |
| 128        | IP Int. Reachability | Provides all the known IP addresses that the given router knows about via one or more internally-originated interfaces. This information may appear multiple times.                                                                                                                                                |
| 129        | Protocols Supported  | Carries the Network Layer Protocol Identifiers (NLPID) for Network Layer protocols that the IS (Intermediate System) is capable. It refers to the Data Protocols that are supported. For example, IPv4 NLPID value 0xCC, CLNS NLPID value 0x81, and/or IPv6 NLPID value 0x8E will be advertised in this NLPID TLV. |
| 130        | IP Ext. Address      | Provides all the known IP addresses that the given router knows about via one or more externally-originated interfaces. This information may appear multiple times.                                                                                                                                                |
| 132        | IP Int. Address      | The IP interface address that is used to reach the next-hop address.                                                                                                                                                                                                                                               |
| 134        | TE Router ID         | This is the Multi-Protocol Label Switching (MPLS) traffic engineering router ID.                                                                                                                                                                                                                                   |
| 135        | TE IP Reachability   | Provides a 32 bit metric and adds a bit for the "up/down" resulting from the route-leaking of L2->L1. Known as the Extended IP Reachability TLV, this TLV addresses the issues with both TLV 128 and TLV 130.                                                                                                      |
| 137        | Dynamic Hostname     | Identifies the symbolic name of the router originating the link-state packet (LSP).                                                                                                                                                                                                                                |
| 10 and 133 |                      | TLV 10 should be used for Authentication; not the TLV 133. If TLV 133 is received, it is ignored on receipt, like any other unknown TLVs. TLV 10 should be accepted for authentication only.                                                                                                                       |


