# F5 BIG-IP & Firewall – Memory Map

> **Goal:** Quick, interview-ready memory map for F5 BIG-IP (LTM/GTM, modules) and Firewalls (stateful/NGFW).

---

## PART 1 – F5 BIG-IP (LOAD BALANCER / ADC)

### 1. Simple idea of a load balancer

- Like a restaurant manager sending customers to different cashiers.
- Sits in front of multiple servers and spreads client requests so no single server is overloaded. [web:2]

**Memory hook:** “Manager at the door → spreads customers to free cashiers.”

---

### 2. What is F5 BIG-IP

- **F5** = company, **BIG-IP** = product family (ADC, security, access). [web:2][web:5]
- Runs **TMOS** (Traffic Management OS); modules (LTM, DNS/GTM, ASM, AFM, APM, etc.) run on top. [web:5]
- Understands apps (HTTP/SSL/DNS), can load-balance, secure, and optimize traffic. [web:2][web:5]

**Memory hook:** “BIG-IP = smart app proxy, not just a load balancer.”

---

### 3. Virtual Server (VS / VIP)

- **Virtual Server (VIP)** = IP:port that clients connect to (front door of F5). [web:5]
- Client never talks directly to backend servers; F5 terminates the connection and picks a server.
- Example: `10.0.0.1:443` as VIP → F5 distributes to `192.168.1.1/2/3:443`.

**Memory hook:** “VIP = one public face, many servers behind.”

---

### 4. Pool and Pool Members

- **Pool** = group of backend servers.  
- **Pool member** = one server (IP:port) inside the pool. [web:5]
- F5 removes a member from rotation automatically when health checks fail.

**Memory hook:** “VIP → Pool → Members (servers).”

---

### 5. Health Monitors

- **ICMP**: ping only, checks reachability.
- **TCP**: checks if port is open.
- **HTTP/HTTPS**: performs HTTP GET, checks status code or string; validates app-level health. [web:5]
- **Custom**: any protocol/URI/response logic.

If monitor fails (timeout, wrong response) → member marked **DOWN**, not used until healthy again. [web:5]

**Memory hook:** “ICMP = alive, TCP = port alive, HTTP = app alive.”

---

### 6. Load Balancing Methods (LTM)

- **Round Robin**: 1,2,3,1,2,3… – equal distribution. [web:4]
- **Least Connections**: chooses server with fewest current connections. [web:4]
- **Weighted / Ratio**: more powerful servers get higher share. [web:4]
- **Fastest**: prefers server with best recent response time. [web:4]
- **Observed / Predictive**: dynamic methods using both load and performance history. [web:4]

**Memory hook:** “Equal (RR), Least busy, Stronger gets more, Faster wins.”

---

### 7. Profiles – the Brain

- Profiles tell BIG-IP how to handle traffic at each layer. [web:5]
- Types:
  - **TCP profile**: timeouts, window/buffer, SYN protection.
  - **HTTP profile**: enables header parsing, rewrites, compression.
  - **Client SSL**: F5 terminates client SSL (offload/inspection). [web:5]
  - **Server SSL**: F5 re-encrypts to servers (SSL bridging). [web:5]
  - **Persistence profile**: controls stickiness.
  - **OneConnect**: reuses fewer server-side TCP connections for many client connections.

**Memory hook:** “No profiles = dumb forwarder; profiles = app-aware proxy.”

---

### 8. Persistence (Stickiness)

- Needed for sessions like carts, logins, WebSockets.
- Types:
  - **Source IP**: same client IP → same server.
  - **Cookie**: F5 inserts cookie with server ID; future requests follow cookie.
  - **SSL session ID**: binds to SSL session.
  - **Universal**: custom key (header, URI, etc.) via iRules.

**Memory hook:** “Same user, same server = persistence.”

---

### 9. SNAT – Source NAT

- Rewrites client source IP with F5 IP so **server always sends replies back via F5**, not directly to client. [web:5]
- Avoids asymmetric routing and broken TCP sessions.
- **SNAT pool** = multiple SNAT IPs to avoid port exhaustion.

**Memory hook:** “SNAT = force all return traffic through F5.”

---

### 10. iRules – F5 Scripting

- TCL-based scripts tied to events like `CLIENT_ACCEPTED`, `HTTP_REQUEST`, `LB_SELECTED`, etc.
- Use cases:
  - Route paths (`/api` → API pool, `/static` → static pool).
  - Insert/remove headers.
  - Block/allow traffic.
  - HTTP→HTTPS redirects.
  - A/B testing, rate limiting.

**Memory hook:** “iRules = programmable brain for each flow.”

---

### 11. iApps and AS3

- **iApps**:
  - GUI-driven templates to deploy an entire app (VS + Pools + Profiles + Monitors) using a wizard. [web:5]
- **AS3**:
  - Declarative JSON: describe desired config (vs, pools, profiles) and BIG-IP builds it.
  - Ideal for CI/CD, GitOps workflows. [web:5]

**Memory hook:** “iApps = templates; AS3 = infra-as-code for BIG-IP.”

---

### 12. GTM / BIG-IP DNS – Global LB

- BIG-IP **DNS (GTM)** = global load balancer using **DNS responses**. [web:9]
- For each DNS query, returns best data center IP based on:
  - Health (DC alive?),
  - Performance (fastest site),
  - Load,
  - Geography. [web:9]
- Uses **Wide IPs** (DNS names mapped to multiple pools/DCs). [web:9]

**Memory hook:** “LTM = inside DC; GTM/DNS = which DC.”

---

### 13. Full Proxy Architecture

- F5 terminates client TCP/SSL, then opens separate TCP/SSL to server. [web:5]
- Client side and server side are independent:
  - Different TCP settings,
  - Different SSL ciphers,
  - F5 can buffer, rewrite, or inspect freely.

**Memory hook:** “Two separate legs: Client–F5, F5–Server.”

---

### 14. SSL Models

- **SSL Offload**:
  - Decrypt at F5, send cleartext to server.
- **SSL Bridging**:
  - Decrypt at F5, inspect/modify, then re-encrypt to server.
- **SSL Passthrough**:
  - F5 does not decrypt; acts as L4 LB only.

**Memory hook:** “Offload (clear to server), Bridge (re-encrypt), Pass (no decryption).”

---

### 15. Security Modules – AFM, ASM/AWAF, APM

- **AFM (Advanced Firewall Manager)**:
  - L3–L4 firewall, ACL, DoS, IP reputation.
- **ASM / Advanced WAF**:
  - L7 Web Application Firewall (OWASP Top 10, bot defense, credential stuffing, L7 DoS).
- **APM (Access Policy Manager)**:
  - SSL VPN, SSO, per-request access policies, MFA integration.

**Memory hook:**  
- AFM = network firewall,  
- ASM/AWAF = app firewall,  
- APM = VPN + access brain.

---

### 16. HA, RHI, ECMP

- **Active/Standby**: one box live, one box idle but synced.
- **Active/Active**: both active, virtuals split into traffic groups.
- **Connection mirroring**: state sync so existing sessions survive failover. [web:5]
- **RHI (Route Health Injection)**:
  - BIG-IP advertises VIPs via OSPF/BGP; routers ECMP load-balance across multiple BIG-IPs. [web:9]

**Memory hook:** “HA for redundancy, RHI+ECMP for scale-out.”

---

### 17. VIPRION & vCMP

- **VIPRION**:
  - Chassis with blades; behaves as one BIG-IP, scales by adding blades.
- **vCMP**:
  - Hypervisor on BIG-IP; splits hardware into multiple virtual BIG-IP guests.

**Memory hook:** “VIPRION = scale up blades; vCMP = carve tenants.”

---

## PART 2 – FIREWALLS

### 21. Simple idea of a firewall

- Like a **bouncer** at a club: checks who’s allowed in/out based on rules.
- Decides **allow/deny** based on policy.

**Memory hook:** “Firewall = rule-based bouncer.”

---

### 22. Packet Filter (Gen 1)

- Stateless: each packet checked standalone (IP, port, protocol).
- Example: router ACLs at the edge.
- Very fast but easy to bypass (no connection awareness).

**Memory hook:** “Packet filter = blind to sessions.”

---

### 23. Stateful Firewall (Gen 2)

- Maintains **state table** (session table).
- Knows NEW/ESTABLISHED/INVALID.
- Allows return traffic automatically for allowed sessions.

**Memory hook:** “Stateful = remembers conversations.”

---

### 24. Zones

- Typical zones:
  - **Inside**: trusted LAN.
  - **Outside**: internet/untrusted.
  - **DMZ**: public servers (web, DNS, mail).
  - **Mgmt**: management-only network.
- Default stance: deny-by-default between zones unless policy allows.

**Memory hook:** “Design zones first, rules second.”

---

### 25. NAT on Firewalls

- **SNAT / PAT**:
  - Many internal hosts share one public IP (outbound internet).
- **DNAT / Static NAT**:
  - Public IP → internal server (e.g., web DMZ).
- **Bidirectional NAT**:
  - Both source and destination NAT; used for overlapping IPs or complex scenarios.

**Memory hook:** “SNAT for users, DNAT for servers.”

---

### 26. NGFW (Gen 3)

- Adds **App-ID** (application awareness), **User-ID** (user identity), **URL filtering**, **IPS**, **file inspection**, **SSL inspection** on top of stateful firewall.
- Can allow/block based on app name and user group, not just IP/port.

**Memory hook:** “NGFW = IP + app + user + content.”

---

### 27. Palo Alto – Zone-based NGFW

- Uses **zone-based policies** with **single-pass engine** (SP3).
- Policy matches: Zone, IP, App-ID, User-ID, Service.
- One unified content engine for IPS, URL filtering, file blocking.

**Memory hook:** “One pass, many checks (SP3).”

---

### 28. Cisco ASA – Classic

- Classic stateful firewall with **security levels** (0–100).
  - Higher → lower allowed by default, reverse blocked unless policy added.
- **MPF** (class-map / policy-map / service-policy) for inspections and limits.
- NGFW features via FirePOWER modules (less integrated than native NGFW).

**Memory hook:** “ASA = stateful core, older NGFW story.”

---

### 29. Cisco FTD – Modern NGFW

- Unified image: ASA + FirePOWER NGFW capabilities.
- Managed by **FMC** (central) or **FDM** (local).
- Native App-ID, URL filtering, IPS, malware protection, SSL inspection.

**Memory hook:** “FTD = ASA + NGFW in one.”

---

### 30. IPS – Intrusion Prevention System

- Inline, blocks attacks:
  - **Signature-based**: known patterns.
  - **Anomaly-based**: deviation from baseline.
  - **Protocol decoders**: detect protocol violations.
- Handles evasion by normalizing fragmented/encoded traffic first.

**Memory hook:** “IPS = pattern + behavior + protocol sanity.”

---

### 31. SSL/TLS Inspection – Forward Proxy

- Firewall acts as **MITM**:
  - Client → FW (internal CA cert),
  - FW → real server.
- Decrypts, inspects, re-encrypts.
- Needs enterprise CA installed on endpoints.
- Limited by **certificate pinning** and **privacy/compliance** rules.

**Memory hook:** “Decrypt–Inspect–Encrypt; watch out for pinning and privacy.”

---

## Ultra-Short Revision Table

| Topic        | One-line memory hook                  |
|-------------|----------------------------------------|
| VIP         | Front door IP/port on F5              |
| Pool        | Group of real servers                 |
| Monitor     | App health checker                     |
| Persistence | Same client, same server              |
| SNAT        | Forces return via F5                  |
| iRules      | Custom traffic logic                  |
| AS3         | JSON-based desired-state config       |
| GTM/DNS     | Picks best DC via DNS                 |
| Stateful FW | Remembers sessions                    |
| NGFW        | IP + app + user + content             |
| SSL inspect | Decrypt → inspect → re-encrypt        |

---

**Tip:**  
- Use this as base in Obsidian.  
- For NotebookLM, break each section (e.g., “VIP”, “Pool”, “SNAT”, “Stateful FW”, “NGFW”) into separate notes to maximize Q&A quality.