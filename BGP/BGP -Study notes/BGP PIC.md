
---

# 🧠 BGP PIC (Prefix Independent Convergence)

👉 PIC is not a new protocol  
👉 It is an **optimization technique inside BGP + FIB**

Goal:

```
Fast convergence WITHOUT BGP full recomputation
```

---

# 🔹 Basic  (Common for Core & Edge)

Normally in BGP failure:

```
Prefix → depends on next-hop
Failure happens →
BGP recalculates per prefix →
FIB updates per prefix
```

Problem:

- Too many prefixes (e.g., Internet = 1M+ routes)
- Convergence becomes slow

👉 PIC changes model:

```
Decouple prefix from next-hop

Prefix → points to indirect next-hop
Next-hop → resolved separately
```

So:

- Failure = update only next-hop
- NOT all prefixes

✅ Result = **massive speed improvement**

---

# 🔹 Two Types of PIC

```
1. BGP PIC Core
2. BGP PIC Edge
```

Think:

|Type|Focus|
|---|---|
|PIC Core|Core link/node failure|
|PIC Edge|Edge PE / eBGP / VPN failure|

---

# 🚀 BGP PIC Core

---

### 🔹 Problem It Solves

In MPLS/IP core:

- Many prefixes depend on SAME next-hop
- If core link/node fails:
    - All prefixes affected
    - Huge recalculation

---

### 🔹 How PIC Core Works

Core idea:

```
Many prefixes → single next-hop → IGP path
```

PIC pre-installs **backup paths in FIB**

Structure:

```
Prefix
  ↓
Next-Hop
  ↓
Primary path + Backup path (IGP/LDP/RSVP)
```

When failure happens:

```
Only next-hop pointer changes
NOT prefix entries
```

---

### 🔹 Example

```
PE1 ---- P1 ---- P2 ---- PE2

Traffic: PE1 → PE2
```

If P1 fails:

Without PIC:

- Recompute all routes via PE2

With PIC:

- Backup path already installed (via alternate IGP path)
- Immediate switch

---

### 🔹 Key Behavior

- Works mainly in **core (P routers)**
- Uses **IGP + LDP/RSVP FRR**
- BGP not heavily involved at failure time

---

### 🔹 What Makes It Fast

Because:

```
BGP table untouched
Only FIB next-hop changes
```

👉 Convergence in milliseconds

---

# 🌍 BGP PIC Edge

---

### 🔹 Problem It Solves

Edge scenario:

- Multiple eBGP peers (Internet/VPN)
- Multiple PEs advertising same prefix

Without PIC:

```
Best path fails →
BGP recomputes →
New path installed →
Delay
```

---

### 🔹 How PIC Edge Works

Idea:

```
Pre-install primary + backup BGP paths
```

Structure:

```
Prefix
  ↓
Multiple next-hops (PE1, PE2)
  ↓
Pre-programmed FIB entries
```

---

### 🔹 Example

```
        RR
       /  \
     PE1  PE2

Both advertise same prefix
```

Without PIC:

- Only best path in FIB

With PIC:

- Both paths pre-installed

Failure:

- Immediate switchover

---

### 🔹 Where It Is Used

- MPLS VPN (multi-homed CE)
- Internet edge (multi upstream ISP)
- Data center fabric (leaf-spine)

---

### 🔹 Dependency

PIC Edge needs:

- Multiple paths visibility ✅ (Add-Path helps here) ✅ (Multipath required)

---

# ⚖️ PIC CORE vs PIC EDGE

|Feature|PIC Core|PIC Edge|
|---|---|---|
|Scope|Core network|Edge (PE/eBGP)|
|Failure type|Link/node|Next-hop/peer|
|Dependency|IGP, FRR|BGP multipath|
|Speed|Very fast|Fast|
|Complexity|Lower|Higher|

---

# 🔥 Deep Design Insight

---

### 🔹 PIC Core is Infrastructure Optimization

- Works silently
- Independent of BGP complexity
- Based on IGP + label switching

👉 “Core heals itself”

---

### 🔹 PIC Edge is Control-Plane Intelligence

- Requires multiple BGP paths
- Depends on RR + Add-Path design
- Needs careful policy planning

👉 “Edge must be prepared”

---

# ⚠️ Where Things Go Wrong

---

### 🔸 Without Add-Path

- RR sends only best path
- PIC Edge has no backup path

👉 PIC Edge becomes ineffective

---

### 🔸 Poor IGP Design

- No alternate path
- PIC Core useless

---

### 🔸 FIB Limitations

- Hardware cannot store backup paths

---

### 🔸 Policy Differences

- Backup path not eligible (policy mismatch)
- Failover fails

---

# 🧠 Memory Map

```
PIC
│
├── Goal
│   └─ Fast convergence without recompute
│
├── Mechanism
│   ├─ Prefix → indirect next-hop
│   └─ Next-hop switch only
│
├── PIC Core
│   ├─ Handles core failures
│   ├─ Uses IGP + labels
│   └─ Very fast
│
├── PIC Edge
│   ├─ Handles edge failures
│   ├─ Needs multiple paths
│   └─ Uses BGP multipath
│
└── Dependency
    ├─ Add-Path (visibility)
    ├─ RR design
    └─ IGP backup paths
```

---

# 🎯 Interview Ready Answer

✅ Short version:

> BGP PIC (Prefix Independent Convergence) is a mechanism that improves convergence speed by decoupling prefix resolution from next-hop resolution. Instead of recalculating every affected prefix during a failure, the router updates only the next-hop, resulting in very fast failover.

✅ Core vs Edge:

> BGP PIC Core handles failures in the core network, such as link or node failures, and relies on IGP and label switching to quickly reroute traffic without BGP recomputation. BGP PIC Edge handles failures at the edge, such as eBGP or PE failures, by pre-installing multiple BGP paths so that traffic can switch instantly to an alternate path without waiting for BGP convergence.

✅ One-liner difference:

> PIC Core is about fast recovery inside the network using IGP, while PIC Edge is about fast recovery at the edge using multiple BGP paths.

---



---

# 🧠 BGP PIC Edge — Mechanisms & Design View

PIC Edge fundamentally requires **multiple usable BGP next-hops pre-installed in FIB**.  
All the methods you listed are simply **different ways to expose multiple paths early**.

👉 Think like this:

```
PIC Edge = Pre-install valid alternate next-hops
Problem = BGP normally hides them
Solution = Use mechanisms to expose them
```

---

# 🔹 Methods Enabling PIC Edge

Below are not independent features — they are **visibility + install enablers**.

---

## 1. BGP Multipath (ECMP)

### Behavior

- Allows installation of **multiple equal-cost BGP paths**
- All next-hops installed in FIB

```
Prefix → NH1 + NH2 (equal cost)
```

### Effect on PIC

- Already active-active
- No recomputation needed

---

### IP Context

- Used in **Internet edge / DC fabrics**
- Multiple eBGP peers → equal AS-PATH / attributes

→ Traffic load-balanced

---

### MPLS Context

- Multiple PE advertising same VPN route
- Label stack programmed per next-hop

→ Fast failover within LSP set

---

✅ Nature:

```
Active-Active model (load sharing)
```

---

## 2. BGP Add-Path

### Behavior

- Allows advertisement of **multiple paths**
- RR does not hide alternate paths

```
RR → advertises PE1 + PE2 both
```

### Effect on PIC

- Enables visibility of backup paths
- Required for PIC in RR-based topologies

---

### IP Context

- Multiple Internet exits visible at edge routers

---

### MPLS Context

- Multiple PE paths for same VPN prefix exposed

---

✅ Nature:

```
Active-Active OR Active-Backup (depends on policy)
```

---

## 3. BGP Diverse-Path

### Behavior

- RR advertises **backup path even if not best**
- Ensures path diversity

```
Best + "as-diverse" alternate path
```

---

### Effect on PIC

- Guarantees at least one alternate path
- Unlike Add-Path → controlled (not all paths)

---

### MPLS Context (Very Important)

- Used in **MPLS VPN**
- Ensures:
    - Primary → one PE
    - Backup → different PE / path

---

✅ Nature:

```
Active-Backup (structured diversity)
```

---

## 4. advertise-best-external

### Behavior

- Router advertises **best external path** even if iBGP best exists

```
iBGP path = best
eBGP path = hidden

→ advertise-best-external exposes eBGP path
```

---

### Effect on PIC

- Makes alternate exit visible
- Important for dual-homed edge

---

### IP Context

- Internet edge routers (dual ISP)

---

### MPLS Context

- Less relevant (more IP-centric)

---

✅ Nature:

```
Active-Backup (external fallback)
```

---

## 5. Add-Path + Multipath Combo (Modern Design)

### Behavior

- Add-Path → visibility
- Multipath → installation

Result:

```
Multiple paths known + multiple paths installed
```

---

### Effect on PIC

👉 This is **ideal PIC Edge design**

---

# 🚀 Active-Active vs Active-Backup — Deep View

---

## 🔵 Active-Active (Load Sharing Model)

### Structure

```
Prefix → NH1 + NH2 (both in FIB)
Traffic → both paths simultaneously
```

---

### Occurs When

- ECMP enabled
- Multipath active
- Equal attributes

---

### IP Example

```
Router → ISP1 + ISP2
Both same cost
→ traffic load shared
```

---

### MPLS Example

```
PE1 → CE (VPN route)
PE2 → CE (same VPN route)

Both installed
→ load-balanced MPLS forwarding
```

---

### Failure Behavior

```
NH1 fails →
Traffic continues via NH2
NO control-plane dependency
```

---

✅ Characteristics:

- Fastest failover
- No traffic disruption
- Needs symmetric paths

---

## 🟠 Active-Backup (Primary/Standby)

---

### Structure

```
Prefix → NH1 (primary) + NH2 (backup pre-installed)
Traffic → NH1 only
```

---

### Occurs When

- Unequal attributes
- Policies prefer one
- Diverse-path used

---

### IP Example

```
Primary ISP (preferred)
Backup ISP (less preferred)
```

---

### MPLS Example

```
Primary PE selected
Backup PE pre-installed via PIC
```

---

### Failure Behavior

```
NH1 fails →
Immediate switch to NH2
(no BGP recompute required)
```

---

✅ Characteristics:

- Traffic engineering control
- Slightly slower than ECMP (but still fast)
- Most common in MPLS VPN

---

# 🔥 IP vs MPLS PIC Edge Thinking

---

## 🌐 IP-Only Networks

Main concern:

- Internet convergence
- Multiple upstream providers

Best combo:

```
Multipath + Add-Path + advertise-best-external
```

Goal:

- Avoid ISP flap impact
- Maintain continuous reachability

---

## 🏭 MPLS Networks (VPN Focus)

Main concern:

- PE failure
- VPN route continuity

Best combo:

```
RR + Add-Path / Diverse-Path + PIC Edge
```

Goal:

- Same prefix from multiple PEs
- Instant VPN failover

---

# ⚖️ Design Mapping (Mental Model)

```
Visibility Layer:
    Add-Path / Diverse-Path / best-external

Installation Layer:
    Multipath / ECMP

Execution Layer:
    PIC Edge (FIB switching)
```

---

# 🧠 Memory Map

```
PIC Edge
│
├── Needs
│   ├─ Multiple path visibility
│   └─ FIB pre-installation
│
├── Methods
│   ├─ Multipath (ECMP)
│   ├─ Add-Path
│   ├─ Diverse-path
│   └─ advertise-best-external
│
├── Models
│   ├─ Active-Active (ECMP)
│   └─ Active-Backup
│
├── IP
│   ├─ Multi-ISP edge
│   └─ Internet routing
│
└── MPLS
    ├─ Multi-PE
    └─ VPN resiliency
```

---

# 🎯 Interview Ready Answer

✅ Crisp answer:

> BGP PIC Edge improves convergence by pre-installing alternate next-hops for a prefix so that upon failure, traffic can switch immediately without waiting for BGP recomputation. It requires mechanisms like multipath, Add-Path, diverse-path, or advertise-best-external to expose multiple paths that BGP normally hides.

---

✅ Active-Active vs Active-Backup:

> In Active-Active, multiple next-hops are installed and used simultaneously via ECMP, providing both load sharing and instant failover. In Active-Backup, one path is active while backup paths are pre-installed and used only upon failure, ensuring controlled traffic engineering with fast convergence.

---

✅ Design insight:

> PIC Edge effectiveness depends on path visibility (Add-Path, diverse-path) and path installation (multipath). Without exposing multiple paths, PIC cannot function properly in RR-based architectures.

---
