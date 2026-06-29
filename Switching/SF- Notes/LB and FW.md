
## PART 1 — LOAD BALANCER (F5 BIG-IP)

---

### WHAT IS A LOAD BALANCER — THE SIMPLE IDEA

Imagine a popular restaurant with one cashier — everyone queues at that one counter and it gets overwhelmed. Now imagine 5 cashiers — a manager at the door directs each customer to the least busy one. That manager is your load balancer. It sits in front of multiple servers and distributes incoming traffic so no single server gets crushed.

---

## BEGINNER LEVEL

*1. What is F5 BIG-IP*
F5 is the company; BIG-IP is their flagship product family. It is an Application Delivery Controller (ADC) — a much smarter version of a basic load balancer. It doesn't just distribute traffic — it understands applications (HTTP, SSL, DNS), manipulates traffic, provides security, and optimizes performance. F5 BIG-IP runs TMOS (Traffic Management Operating System) as its base OS. Everything runs as a module on top of TMOS.

*2. Virtual Server (VS)*
The front door of F5. A Virtual Server is an IP address and port combination that F5 listens on — clients connect to this IP, not directly to backend servers. F5 receives the connection, makes a load balancing decision, and forwards to a real server behind it. The client never knows which real server handled their request. Example: VS IP is 10.0.0.1:443 — all HTTPS traffic hits this, F5 distributes to web servers 192.168.1.1, .2, .3 behind it.

*3. Pool and Pool Members*
A Pool is a group of backend servers (called pool members) that the virtual server distributes traffic to. Each pool member has an IP and port. Pool members can be marked up/down based on health monitors. If a member fails its health check, F5 removes it from rotation automatically — clients never hit a dead server.

*4. Health Monitors*
F5 constantly checks if pool members are alive using monitors:
- *ICMP*: just pings the server — basic, only checks network reachability
- *TCP*: checks if the port is open — better, but doesn't verify the app works
- *HTTP*: sends an actual HTTP GET request and checks for a specific response code or string — confirms the app is responding correctly
- *HTTPS*: same as HTTP but over SSL
- *Custom*: send any request, match any response — used for database checks, API health endpoints

If a monitor fails (no response within timeout, wrong response), the pool member is marked down and removed from load balancing until it recovers.

*5. Load Balancing Methods*
How F5 decides which pool member gets the next connection:
- *Round Robin*: sends to each server in turn — 1,2,3,1,2,3. Simple, good when all servers are equal
- *Least Connections*: sends to whichever server has fewest active connections — better for varying request lengths
- *Weighted Round Robin*: gives more traffic to more powerful servers (Server A gets 3x traffic of Server B)
- *Fastest*: sends to the server that responded fastest recently — good for latency-sensitive apps
- *Ratio*: static weight-based distribution
- *Observed / Predictive*: dynamic algorithms that track both connections and response times together

---

## INTERMEDIATE LEVEL

*6. Profiles — The Brain of F5*
Profiles tell F5 how to handle traffic at each layer. Without profiles, F5 is just a basic forwarder. With profiles, it becomes an application-aware proxy.

- *TCP Profile*: controls TCP behavior — connection timeout, keep-alive, buffer sizes, SYN cookie protection. You can have a different TCP profile for client-side and server-side — this is called TCP Express or OneConnect optimization
- *HTTP Profile*: enables HTTP parsing — header insertion, compression, cookie handling, redirect rewriting, request/response manipulation
- *SSL Profile (Client SSL)*: F5 terminates SSL from clients — decrypts traffic, inspects it, re-encrypts (or sends clear) to backend. Essential for visibility and WAF
- *SSL Profile (Server SSL)*: F5 re-encrypts traffic toward backend servers — end-to-end SSL with F5 in the middle doing inspection
- *Persistence Profile*: ensures a client always goes to the same server (covered next)
- *OneConnect Profile*: pools TCP connections to backend servers — reuses a smaller number of server-side connections for a larger number of client connections, reducing server TCP overhead dramatically

*7. Persistence*
Some applications require a user to always hit the same backend server — shopping carts, session-based apps, WebSocket connections. This is called persistence or session affinity. Methods:
- *Source IP Persistence*: same client IP always goes to same server — simple but breaks with NAT (many users behind one IP)
- *Cookie Persistence*: F5 inserts a cookie in the HTTP response with the server ID — subsequent requests carry the cookie, F5 reads it and sends to same server. Most reliable for HTTP apps
- *SSL Session ID*: uses SSL session ID to pin client to server — useful when HTTP visibility isn't available
- *Universal Persistence*: persist on any custom field using iRules — very flexible

*8. SNAT — Source Network Address Translation*
When F5 forwards traffic to a pool member, the pool member needs to return traffic back through F5 (not directly to the client) — otherwise the TCP handshake breaks. SNAT makes F5 replace the client's source IP with F5's own IP before sending to the backend — so the server always sends return traffic to F5. Without SNAT, servers send replies directly to clients, bypassing F5, causing connection resets. SNAT Pool: F5 uses a pool of IPs for translation — important when traffic volume exceeds what a single IP can handle (port exhaustion limit ~65535 concurrent connections per SNAT IP).

*9. iRules — F5 Scripting Language*
iRules is F5's Tcl-based scripting language — allows custom traffic manipulation logic triggered by events in the connection/request lifecycle. Events: CLIENT_ACCEPTED, HTTP_REQUEST, HTTP_RESPONSE, LB_SELECTED, SSL_CLIENTHELLO, etc.

Examples of what iRules can do:
- Route traffic to different pools based on URL path (/api → API pool, /static → CDN pool)
- Insert/remove HTTP headers
- Block traffic based on IP, header value, or URI pattern
- Redirect HTTP to HTTPS
- Rate limit connections per client IP
- Perform A/B testing by sending % of traffic to new pool

iRules execute at line rate in hardware — extremely powerful but require careful coding as bugs can crash connections.

*10. iApps and AS3 — Declarative Configuration*
- *iApps*: application templates that guide F5 configuration via wizards — deploy a complete application service (VS + Pool + Profiles + Monitors) from a single template
- *AS3 (Application Services 3 Extension)*: modern declarative API — describe the desired end state in JSON, F5 figures out how to configure itself. No need to configure object-by-object. Integrates with CI/CD pipelines for automated F5 deployment. CCDE-level: AS3 is the foundation of F5's automation story

*11. GTM / DNS — Global Traffic Manager*
F5 GTM (now called BIG-IP DNS) provides global load balancing across multiple data centers using DNS. When a client queries DNS, GTM responds with the IP of the best data center based on:
- Geographic proximity (closest DC)
- Health of the DC (don't send to a DC where the app is down)
- Load (send to the less loaded DC)
- Performance (send to the fastest-responding DC)

GTM uses Wide IPs — a DNS name mapped to multiple pools in different DCs. LTM (Local Traffic Manager) handles traffic within a single DC; GTM handles traffic between DCs. Together they provide multi-site active-active or active-standby load balancing.

---

## ADVANCED LEVEL

*12. Full Proxy Architecture*
F5 operates as a full proxy — it terminates the client connection completely and establishes a separate connection to the backend server. This means: client and server have entirely independent TCP stacks, TCP parameters, and SSL sessions. Benefits: F5 can buffer slow clients without tying up server connections, apply independent QoS policies, rewrite application content, and hide server-side errors. This is fundamentally different from a simple NAT/DSR (Direct Server Return) load balancer which just redirects packets without terminating connections.

*13. SSL Offload and SSL Bridging vs SSL Passthrough*
Three SSL models:
- *SSL Offload*: F5 decrypts client SSL, sends plaintext to backend — server has zero SSL CPU burden. Backend must be in a trusted network
- *SSL Bridging (Re-encryption)*: F5 decrypts from client, inspects/manipulates, re-encrypts to server — full visibility and security, end-to-end encryption, higher F5 CPU cost
- *SSL Passthrough*: F5 passes encrypted traffic straight through to backend without decryption — no visibility into application layer, persistence limited to IP-based, no WAF possible. Used when compliance requires data never decrypted in transit

*14. AFM — Advanced Firewall Manager*
F5's L4 firewall module. Provides network-layer protection in front of applications:
- ACL-based rules (source IP, destination IP, port, protocol)
- DoS protection (SYN flood, UDP flood, ICMP flood) with hardware assist
- IP intelligence (reputation-based blocking using threat feeds)
- Port scanning detection and blocking
- Protocol anomaly detection

Operates before LTM processing — malicious L4 traffic blocked before it reaches virtual server processing, reducing CPU load.

*15. ASM / AWAF — Application Security Manager / Advanced WAF*
F5's Layer 7 Web Application Firewall. Protects against OWASP Top 10 and beyond:
- *Positive security model*: only allow known-good traffic (whitelist) — strict but requires learning period
- *Negative security model*: block known-bad patterns (signatures) — easier to deploy, less effective against zero-days
- *Automatic Policy Builder*: F5 learns normal application traffic in transparent mode, builds a baseline policy, then switches to blocking mode
- *Bot Defense*: distinguishes human users from bots — browser challenges, JavaScript injection, fingerprinting
- *Credential Stuffing Protection*: detects login attacks using stolen credential databases
- *L7 DoS*: rate-limits HTTP requests per URI, per source IP, per session — blocks slowloris and HTTP flood attacks
- *DataSafe*: client-side encryption of form fields — protects against keyloggers and Man-in-the-Browser attacks on the endpoint

*16. APM — Access Policy Manager*
F5's SSL VPN and identity/access control module:
- SSL VPN: remote users connect to F5, APM authenticates and provides network access
- Per-request policy: each HTTP request evaluated for access — not just at session start
- SSO (Single Sign-On): F5 handles authentication once, passes identity headers to backend apps — apps don't need their own auth stacks
- MFA integration: RADIUS, LDAP, SAML, OAuth — APM integrates with all major identity providers
- Clientless access: browser-based access to internal apps without VPN client
- Network Access (full tunnel) vs Web Access (reverse proxy) modes

*17. BIG-IP HA — Active/Standby and Active/Active*
F5 HA uses Config Sync and Device Service Clustering (DSC):
- *Active/Standby*: one F5 active, one standby — standby syncs config and connection table via HA VLAN. Failover in ~3 seconds. Standby is completely idle
- *Active/Active*: both F5s active, each handling different traffic groups (virtual servers split between devices) — better utilization, but failover means surviving device handles all traffic (must be sized for this)
- *Connection mirroring*: active device mirrors connection state to standby — on failover, existing TCP/SSL sessions survive. Without mirroring, all active connections reset on failover

*18. ECMP and Route Health Injection (RHI)*
F5 advertises Virtual Server IPs into the routing domain via RHI (using OSPF or BGP) — routers learn F5's VS IPs as host routes and route traffic to F5. In multi-F5 environments, RHI enables ECMP — multiple F5s advertise the same VS IP, router distributes traffic across all of them using ECMP hashing. This is how enterprise-scale F5 deployments achieve horizontal scaling beyond a single HA pair. CCDE must design ECMP hash alignment between router and F5 to avoid uneven distribution.

*19. iCall and iStats — Operational Monitoring*
- *iCall*: F5's built-in event/automation framework — trigger scripts based on events (pool member down, connection threshold exceeded). Enables self-healing behaviors without external automation
- *iStats*: custom statistics tracking within iRules and iCall — track per-URI hit counts, per-client connection rates, custom business metrics exposed via SNMP or REST API

*20. BIG-IP VIPRION and vCMP*
*VIPRION*: F5's chassis-based platform — blade-based modular system. Multiple blades in a chassis share a backplane, appearing as a single logical F5. Blades can be added for scale. Chassis provides high-bandwidth non-blocking fabric between blades.

*vCMP (Virtual Clustered Multiprocessing)*: hypervisor built into F5 — allows partitioning a physical F5 (or VIPRION chassis) into multiple isolated virtual F5 guests. Each guest has dedicated CPU, memory, and SSL resources. Enables multi-tenancy on a single physical platform — different business units or customers get their own isolated F5 instance sharing the same hardware.

---

## PART 2 — FIREWALL

---

### WHAT IS A FIREWALL — THE SIMPLE IDEA

A bouncer at a nightclub checks IDs, has a guest list, and decides who gets in and who doesn't. A firewall is that bouncer for your network — it inspects traffic and decides what's allowed and what's blocked based on rules.

---

## BEGINNER LEVEL

*21. Packet Filter Firewall (Generation 1)*
The simplest type — looks at each packet individually (source IP, destination IP, protocol, port) and matches against a rule list. Stateless — doesn't know if a packet belongs to an existing connection or is a new one. Very fast but easily bypassed — an attacker can forge source IPs or use allowed ports to sneak malicious traffic. Access Control Lists (ACLs) on routers are essentially packet filters. Still used at network edge for bulk IP blocking.

*22. Stateful Firewall (Generation 2)*
Tracks the state of every connection in a state table (also called connection table or session table). Knows whether a packet is part of an established session, a new connection attempt, or an invalid/spoofed packet. Allows return traffic for established connections automatically — you only need a rule for the initiating direction. Blocks packets that don't match a known session state — defeating most IP spoofing attacks. This is the baseline for all modern firewalls.

*23. Firewall Zones*
Firewalls divide the network into security zones:
- *Inside (Trusted)*: internal corporate network
- *Outside (Untrusted)*: internet
- *DMZ (Demilitarized Zone)*: semi-trusted — public-facing servers (web, email, DNS) that need internet access but shouldn't have full internal access
- *Management Zone*: out-of-band management network

Traffic between zones is controlled by policies. Default behavior: deny all between zones unless explicitly permitted. Zone design is the foundation of network security architecture.

*24. NAT on Firewalls*
Firewalls also perform NAT (Network Address Translation):
- *SNAT / PAT (Port Address Translation)*: many internal IPs share one public IP — the firewall tracks which internal host made each connection using port numbers. Standard internet access for most organizations
- *DNAT / Static NAT*: maps a public IP to a specific internal server — allows internet users to reach internal servers (web servers in DMZ)
- *Bidirectional NAT*: both source and destination NAT applied simultaneously — used in complex overlapping address scenarios

---

## INTERMEDIATE LEVEL

*25. NGFW — Next Generation Firewall (Generation 3)*
Traditional stateful firewalls only understand IP, port, and protocol. NGFWs add:
- *Application Identification (App-ID)*: identifies the application regardless of port — YouTube on port 443 is identified as YouTube, not just HTTPS. Can block specific apps while allowing others on same port
- *User Identity (User-ID)*: maps traffic to specific users via Active Directory integration — policy based on user/group, not just IP
- *SSL Inspection*: decrypts and inspects SSL/TLS traffic — without this, NGFWs are blind to ~90% of modern traffic
- *IPS (Intrusion Prevention System)*: inline threat detection and blocking — signature-based and anomaly-based detection
- *URL Filtering*: categorizes and controls web access by URL category (social media, malware, gambling)
- *File Inspection*: inspects file types in transit — blocks malicious file types or sandboxes unknown files

*26. Palo Alto Architecture — Zone-Based NGFW*
Palo Alto is the most common NGFW at CCIE/CCDE level. Key architectural concepts:
- *Single-pass parallel processing (SP3)*: performs all functions (App-ID, User-ID, Content-ID, threat inspection) in a single pass through the data plane — unlike traditional firewalls that chain multiple engines
- *App-ID engine*: uses application signatures, protocol decoding, and behavior analysis to identify apps
- *Content-ID engine*: single engine handles IPS, URL filtering, file blocking, data filtering
- *Security policies*: match on Zone + Source/Destination IP + Application + User + Service — most granular policy model in the industry

*27. Cisco ASA — Classic Stateful Firewall*
Cisco's traditional firewall — still widely deployed. Key concepts:

- *Security levels*: interfaces have security levels (0–100). Higher security can initiate to lower by default; return traffic automatically permitted. Outside = 0, Inside = 100, DMZ = 50
- *MPF (Modular Policy Framework)*: class-map → policy-map → service-policy structure for QoS, connection limits, inspection policies
- *ASA with FirePOWER*: adds NGFW capabilities (IPS, App-ID, URL filtering) as a module — but less integrated than native NGFW platforms

*28. Cisco FTD — Firepower Threat Defense*
Cisco's modern NGFW — merges ASA stateful firewall with FirePOWER NGFW features into a single unified image. Managed by FMC (Firepower Management Center) for centralized policy management. FDM (Firepower Device Manager) for standalone management. Key advancement over ASA: native App-ID, URL filtering, IPS, malware protection, and SSL inspection are first-class features rather than add-ons.

*29. IPS — Intrusion Prevention System*
IPS sits inline and inspects traffic for attack patterns:
- *Signature-based*: matches traffic against a database of known attack patterns — fast, reliable for known threats, blind to zero-days
- *Anomaly-based*: establishes a baseline of normal behavior, alerts on deviations — catches unknown threats but produces more false positives
- *Protocol decoding*: verifies traffic conforms to protocol standards — catches protocol abuse attacks
- *Evasion techniques and countermeasures*: attackers fragment packets, use encoding, or send out-of-order segments to bypass signature matching — modern IPS normalizes traffic before inspection

*30. SSL/TLS Inspection — Forward Proxy*
Without SSL inspection, a firewall sees an encrypted tunnel and can't inspect contents — attackers use HTTPS to hide malware and C2 traffic. SSL inspection (also called SSL decryption or man-in-the-middle inspection):

1. Client connects to firewall — firewall presents a certificate signed by an internal CA
2. Firewall connects to destination server and gets real certificate
3. Firewall decrypts, inspects, re-encrypts in both directions

Requires deploying a trusted internal CA certificate to all clients (via GPO). Certificate pinning breaks SSL inspection — some apps (banking, OS updates) bypass inspection via certificate pinning exceptions. Privacy concerns: all user HTTPS traffic visible to firewall — compliance and legal considerations important.