### 1. VRRP (Virtual Router Redundancy Protocol)

|Aspect|Details|
|---|---|
|**What it is**|An **open standard** (IETF RFC 5798) protocol that provides **gateway redundancy** by allowing multiple routers to share a **virtual IP (VIP)** and **virtual MAC**. One router is **Active (Master)**, others are **Backup**.|
|**Why it is required (Use Case)**|- **High availability** for default gateway in LANs. - End hosts configure **one default gateway (VIP)**; no need to update hosts if physical router fails. - **Vendor interoperability** (Cisco, Juniper, Arista, etc.). - Preferred in **multi-vendor environments** or **open-standard deployments**.|
|**How it works**|1. Routers in VRRP group share a **Virtual IP** and **Virtual MAC (00-00-5E-00-01-{VRID})**. 2. **Master** (highest priority, default 100) owns VIP and responds to ARP. 3. Master sends **multicast advertisements (224.0.0.18)** every 1 sec (default). 4. If Master fails, highest-priority Backup **preempts** and becomes Master. 5. **Preemption** enabled by default (configurable).|

> **Key Notes**:
> 
> - Priority: 1–255 (255 = highest)
> - Authentication: MD5 (deprecated in RFC 5798)
> - Supports **IPv4 and IPv6 (VRRPv3)**

---

### 2. HSRP (Hot Standby Router Protocol)

|Aspect|Details|
|---|---|
|**What it is**|**Cisco-proprietary** FHRP that provides gateway redundancy using **Active/Standby** model with a **virtual IP and MAC**.|
|**Why it is required (Use Case)**|- Legacy Cisco networks requiring **seamless failover**. - Simple configuration, widely supported in **Cisco-only environments**. - Used when **load balancing is not needed** (only one router active).|
|**How it works**|1. Two or more routers form an **HSRP group** (0–255). 2. **Active** router (highest priority) owns **VIP** and **MAC (0000.0C07.ACxx)**. 3. Active sends **Hello messages (multicast 224.0.0.2)** every 3 sec. 4. **Standby** monitors; takes over if no Hellos in **hold time (10 sec)**. 5. **Preemption** must be explicitly enabled.|

> **Key Notes**:
> 
> - Priority: 0–255 (default 100)
> - Supports **interface tracking** (reduce priority on uplink failure)
> - **HSRPv2** adds IPv6 and millisecond timers

---

### 3. GLBP (Gateway Load Balancing Protocol)

|Aspect|Details|
|---|---|
|**What it is**|**Cisco-proprietary** FHRP that provides **redundancy + load balancing** by assigning different **virtual MACs** to hosts via **ARP responses**.|
|**Why it is required (Use Case)**|- **Full utilization of multiple uplinks** (both routers forward traffic). - Ideal for **high-throughput environments** (e.g., data centers, campus core). - Eliminates **single active router bottleneck** in HSRP/VRRP.|
|**How it works**|1. **AVG (Active Virtual Gateway)** owns VIP and assigns **virtual MACs** to **AVFs (Active Virtual Forwarders)**. 2. Up to **4 AVFs** per group; each has unique **virtual MAC (0007.B4xx.xxxx)**. 3. Hosts ARP for VIP → AVG replies with **different virtual MACs** (round-robin, weighted, or host-dependent). 4. Traffic is **load-balanced** across routers. 5. If AVG fails, highest-priority backup becomes AVG.|

> **Key Notes**:
> 
> - **Load balancing methods**: Round-robin (default), weighted, host-dependent
> - **Preemption** for AVG and AVF roles
> - Supports **interface tracking**