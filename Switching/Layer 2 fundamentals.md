# STP (Spanning Tree Protocol) - Memory Map

> **Quick Goal:** Remember STP port roles, topology change flow, and key features in 5 minutes

---

## 1. STP Port Roles (Memory Hook)

| Role | Where | Key Function |
|------|-------|--------------|
| **Root Port (RP)** | Non-root switch | Best path to root (lowest cost) |
| **Designated Port (DP)** | All switches | Sends BPDUs downstream |
| **Blocked Port** | Non-root switch | Backup, listens only |

**Memory Map:** `Root Bridge → All DPs → Non-root has 1 RP → Rest blocked`
[[STP_Port Roles.png]]
---

## 2. Normal STP Operation

✅ Non-root switch **receives BPDUs on Root Port**  
❌ Non-root switch **never sends BPDUs to root bridge**  
📉 BPDUs flow **downstream** from root

---

## 3. Topology Change Flow (TCN → TCA → TC)

### Memory Chain (4 Steps)

**Step 1:** Local Switch detects TC  
→ Generates **TCN BPDU**  
→ Sends on **Root Port → toward root**

**Step 2:** Upstream Switch receives TCN  
→ Sends **TCA (ack)** on **Designated Port → downstream**  
→ Creates **new TCN** on **Root Port → up the tree**

**Step 3:** Repeat until Root Bridge  
→ TCN travels **up the tree**

**Step 4:** Root Bridge receives TCN  
→ Sets **TC bit = 1** in BPDUs  
→ Sends to **all switches** (forwarding + blocked)  
→ Switches **reduce MAC aging time**

**Visual:** `Local → TCN → Upstream → TCA(down)+TCN(up) → ... → Root → TC bit → All`

---

## 4. STP Selection Rules (4 "Lowest" Rules)

| Priority | Rule | Purpose |
|----------|------|---------|
| 1 | **Lowest Bridge ID** | Becomes **Root Bridge** |
| 2 | **Lowest Path Cost** | Selects **Root Port** |
| 3 | **Lowest Sender Bridge ID** | Tiebreak: different upstream switches |
| 4 | **Lowest Sender Port ID** | Tiebreak: same switch, same cost |

**Memory Hook:** `Bridge ID → Cost → Sender BID → Sender Port ID`

---

## 5. Key STP Features

### 🔥 PortFast

| Aspect | Behavior |
|--------|----------|
| Purpose | Access port → **immediate forwarding** (skip Learn/Listen) |
| TCN | **No TCN BPDU generated** |
| Use Case | Servers/PCs joining/leaving (avoid 2×·FW delay + TC flood) |
| BPDU | Active if **no BPDU**; turns **OFF** if BPDU received |

**Memory:** `PortFast = Instant Forward + No TCN = Avoid Blockage for Access Ports`

---

### ⚡ BackboneFast

| Aspect     | Behavior                                                 |
| ---------- | -------------------------------------------------------- |
| Trigger    | **Inferior BPDU** on **blocked port** (indirect failure) |
| Action     | Sends **RLQ → Root**                                     |
| Root Reply | Sends **RL Reply**                                       |
| Result     | Port → forwarding in **2×·FWD** (skips 20s max-age)      |

**Memory:** `BackboneFast = Inferior BPDU on Blocked → RLQ → Skip Max-Age → 2×·FWD`

---

### 🛡️ BPDU Guard vs BPDU Filter

| Feature | Global Mode | Interface Mode | Incoming BPDU | Outgoing BPDU |
|---------|-------------|----------------|---------------|---------------|
| **BPDU Guard** | With PortFast | — | **Port inconsistent**, PortFast OFF | Blocked |
| **BPDU Filter (Global)** | With PortFast | — | PortFast first → only **outgoing** | filtered |
| **BPDU Filter (Int)** | — | **Always** | filtered | filtered |

**Key Rules:**

1. **BPDU Guard (Global):** Handles incoming BPDU **first** → Port inconsistent, PortFast disabled

2. **BPDU Filter (Global):** PortFast handles incoming first → If PortFast OFF → only **outgoing** filtered

3. **BPDU Filter (Interface):** Filters **all BPDUs** (ignores PortFast)

**Flow:** `Guard → blocks incoming + disables PortFast`  
`Filter (Global) → PortFast first → only outgoing`  
`Filter (Int) → all BPDU filtered`

---

## 6. One-Liner Memory Summary

| Concept              | One-Liner                          |
| -------------------- | ---------------------------------- |
| Root Bridge          | Lowest Bridge ID                   |
| Root Port            | Lowest cost to root                |
| TCN Flow             | Up to root → TC bit → all switches |
| PortFast             | Instant forward, no TCN            |
| BackboneFast         | Inferior BPDU → RLQ → skip max-age |
| BPDU Guard           | Incoming BPDU → port inconsistent  |
| BPDU Filter (Global) | Only outgoing (if PortFast active) |
| BPDU Filter (Int)    | All BPDUs filtered                 |

---

## 7. Diagram References

- Port Roles: [[`STP_Port_Roles.png`]]
- TCAck: `TCAck.png`
- TC Notification: `TC_Notify.png`
- BackboneFast: `backbone.png`

---

**💡 Tip:** Read this daily for 3 days → STP concepts will stick!


 








