!
!
==Idle==                  Indicates the router is currently not attempting any connection establishments.

==Connect== -         Indicates the router is waiting for the TCP connection to be completed. If successful  
						an OPEN message is sent

==Active==              Indicates the router didn't receive agreement on parameters of establishment and is  
						trying to initiate TCP.

==OpenSent== -      After the TCP session is setup, the router waits for an OPEN message to confirm all 
						parameters.
						If no errors a BGP keepalive message is sent.

==OpenConfirm== - Indicates the router is waiting for a keepalive or notification message.
						If a keepalive is received the state Changes to Established, else Changes to Idle

==Established== - Indicates peering to a neighbor is established; routing begins


|             |                                                                                |                                                                                                                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BGP State   | Description                                                                    | Causes of Being Stuck                                                                                                                                                                                                                                                                                                                          |
| Idle        | Initial state; BGP waits to initiate TCP connection, no resources allocated.   | 1. Incorrect neighbor IP/ASN (e.g., neighbor 192.168.1.1 remote-as 65001 but peer uses 65002). 2. No route to peer in RIB (missing IGP/static route). 3. BGP process down (resource exhaustion, software bug). 4. ACL/CoPP blocking TCP port 179. 5. Peer lacks matching neighbor statement. 6. BGP suppression (e.g., bgp enforce-first-as ). |
| Active      | BGP initiated TCP connection, but handshake incomplete (SYN sent, no SYN-ACK). | 1. Peer BGP process down or missing neighbor statement. 2. Firewall/ACL dropping TCP 179 packets. 3. Path MTU mismatch (e.g., GRE/IPsec tunnel fragmentation). 4. Network congestion/packet loss. 5. Incorrect update-source (e.g., physical vs. loopback IP). 6. MD5 password mismatch.                                                       |
| Connect     | BGP awaiting TCP handshake completion (SYN sent, awaiting SYN-ACK).            | 1. Peer not listening on TCP 179 (BGP down or misconfigured). 2. Network issues (latency, packet loss). 3. Firewall/CoPP dropping SYN packets. 4. Unexpected source IP (NAT or wrong update-source ). 5. Router resource constraints (high CPU/memory).                                                                                        |
| OpenSent    | TCP established; BGP OPEN sent, awaiting peer’s OPEN message.                  | 1. Mismatched BGP parameters (ASN, BGP version). 2. Unacceptable hold time in OPEN message. 3. Router ID conflict (same ID on both peers). 4. MD5 authentication mismatch. 5. Peer not sending OPEN (misconfiguration/software bug). 6. Packet loss/corruption.                                                                                |
| OpenConfirm | OPEN received/validated; KEEPALIVE sent, awaiting peer’s KEEPALIVE.            | 1. Peer not sending KEEPALIVE (misconfiguration/bug). 2. Network packet loss/corruption. 3. Timer mismatch (hold time/keepalive interval). 4. Firewall dropping KEEPALIVE packets. 5. BGP software bug (e.g., IOS/JunOS defect).                                                                                                               |