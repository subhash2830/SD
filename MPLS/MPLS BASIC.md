!
How label distribution can be done
				Piggyback the labels on existing IGP
				Use new protocol for label exchange(LDP)


different fields of MPLS label
				Actual label - 20 bits
				Experimental bits - 3 bits  : used for quality of service
				Bottom of stack - 1 bit  :: if set to 1 means this is the last label in label stack
				TTL - 8 bits                :: To avoid packet being stuck in routing loop
			    Range :
			    0 - 15 Reserved
                16 - 2^ 20 - 1 unreserved

different label operations
			Push - add label and forward packet
			Swap - replace label and forward packet
			Pop - remove the label and forward packet
			Untagged - remove all label and forward unlabeled packet
			Aggregate - remove all label do IP lookup and forward packet

different label space
			per-interface (ATM interface)
			per-platform (non ATM interface)

different MPLS modes
			Label distribution mode 
						Downstream on demand( atm interface)
						unsolicited downstream( non atm interface)
			Label retention mode
						Liberal ( used by non atm )
			            Conservative ( used by atm)
			LSP control mode
						independent 
						ordered

different reserved labels
			Implicit Null - label 3
			Explicit Null - label 0 and 2 for ipv6
			Router alert - label 1
			OAM alert - label 14

benefits of MPLS
			BGP free core
			MPLS vpn L2 and L3
			MPLS TE

protocols used for label exchange
		LDP - used for IGP prefix label exchange
		BGP - for BGP prefic label exchange
		RSVP - used for MPLS TE related labels

what is LSP
         LSP is sequence of LSR that switch the packets through MPLS network

ingress LSR
> Ingress LSRs receive a packet that is not labeled yet, insert a label (stack) in front of the packet, and send it on a data link. One that is doing imposition is an ingress LSR


What is Egress LSR 

 > Egress LSRs receive labeled packets, remove the label(s), and send them on a data link. Ingress and egress LSRs are edge LSRs. One that does disposition is an egress LSR
 
Why MPLS needed
### **Why MPLS is REQUIRED – 6 TAC-Level Reasons**
> **MPLS is a **label-switching forwarding paradigm** that decouples **routing (control plane)** from **forwarding (data plane)** using **32-bit labels** — required for **VPNs, Traffic Engineering (TE), QoS, Fast Reroute (FRR), and scalability** in service provider and enterprise networks.”**

MPLS is required because IP routing alone cannot do VPNs, TE, or sub-50ms FRR. It uses labels to forward, BGP to signal VPNs, and LDP/RSVP for transport — enabling scalable, secure, high-performance services in SP and enterprise networks.”

| **#** | **Requirement**                   | **How MPLS Solves**              | **Without MPLS**            |
| ----- | --------------------------------- | -------------------------------- | --------------------------- |
| 1     | **L3VPN (Customer Isolation)**    | **VPNv4 + RD/RT** over MPLS      | Overlapping IPs collapse    |
| 2     | **Traffic Engineering (TE)**      | **RSVP-TE + Explicit Path**      | IGP = shortest path only    |
| 3     | **Fast Reroute (FRR)**            | **LDP/TE FRR < 50ms**            | IGP reconvergence = 100s ms |
| 4     | **QoS (Class-Based Forwarding)**  | **EXP bits → Queue mapping**     | IP Precedence limited       |
| 5     | **Scalability (PE-only Routing)** | **P routers NO customer routes** | Full mesh IGP = n²          |
| 6     | **Layer 2 Services (L2VPN)**      | **VPWS, VPLS, EVPN** over MPLS   | No Ethernet over IP         |