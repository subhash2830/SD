## 🔹 What Problem Does It Solve?

- **Challenge****:** LDP required a full mesh of targeted sessions, each PE advertising unique service labels per remote PE.
    
- **Solution****:** With BGP handling **both autodiscovery and signaling**, we eliminate LDP entirely.
    
- **Design impact:** One protocol (BGP) manages membership and label signaling, simplifying the control plane.
    

## 🔹 Why Use BGP‑Only?

- **Operational simplicity****:** No LDP sessions to configure or troubleshoot.
    
- **Scalability****:** Route Reflectors distribute VPLS routes; adding new PEs requires no manual mesh updates.
    
- **Efficiency****:** Label blocks avoid per‑PE updates, reducing BGP churn.
    
- **Design driver****:** Ideal when the network already runs large‑scale iBGP and wants to avoid LDP complexity.
    

## 🔹 How It Works

- **BGP AFI/SAFI****:** Each PE advertises one update per VPLS domain.
    
- **Label blocks****:** Instead of per‑PE labels, each PE advertises a block of labels.
    
    - Remote PEs calculate which label to use via a formula:
        

Block Offset=round(VE IDBlock Size)×Block Size

- Ensures uniqueness without explicit per‑PE signaling.
    
- **VE ID****:** Unique per PE, used in the formula to select labels.
    
- **RD/RT/AGI****:**
    
    - RD → uniqueness per PE.
        
    - RT → membership control.
        
    - AGI → identifies VPLS instance.
        
- **Data plane****:** Same as before — bridging, MAC learning, split horizon.

[[VPLS_BGP_only.txt 1]]
## Interview‑Ready Summary

> “In a BGP‑only VPLS design, BGP handles both autodiscovery and service label signaling. Each PE advertises one update per VPLS domain containing a block of labels. Remote PEs use a formula based on VE IDs to select the correct label from the block, ensuring uniqueness without explicit per‑PE signaling. The Route Reflector distributes these updates, and the data plane remains identical to traditional VPLS — bridging, MAC learning, and split horizon. This design is chosen when scalability and operational simplicity are priorities, especially in networks already running large‑scale iBGP.”

## Memory Map (Quick Recall)

Code

```
VPLS → L2VPN → SP core = distributed switch
   ├─ Hybrid (BGP AD + LDP signaling)
   └─ BGP-only
        ├─ BGP handles autodiscovery + signaling
        ├─ RD → uniqueness
        ├─ RT → membership
        ├─ AGI → VPLS instance ID
        ├─ VE ID → unique per PE
        ├─ Label blocks → formula-based allocation
        └─ Service label signaled via BGP
Data plane unchanged → bridging, MAC learning, split horizon
Verification → EIGRP neighbors across VPLS
```


# 📘 Simplified VPLS with BGP‑Only (Label Signaling)

## 🔹 The Change

- **Before****:** LDP handled signaling. Each PE had a point‑to‑point session with every other PE, and automatically assigned a **unique service label per remote PE**.
    
- **Now****:** BGP handles signaling. No LDP sessions.
    

## 🔹 The Problem

- In VPLS, each PE must know **which remote PE a frame came from** (for MAC learning).
    
- With LDP, this was easy: each pseudowire had its own label.
    
- With BGP, if every PE advertised only one label for the whole VPLS, the local PE couldn’t tell which remote PE sent the frame.
    

## 🔹 The Solution

- Instead of sending one update per remote PE (which doesn’t scale), BGP uses a **label block mechanism**.
    
- **Each PE advertises one update per VPLS domain**.
    
- That update contains:
    
    - **VE ID****:** Unique per PE.
        
    - **Block offset****:** Where the block starts.
        
    - **Block size****:** How many labels are in the block.
        
    - **Label base****:** Starting number for the block.
        
- **Formula:**
    

Block Offset=round(VE IDBlock Size)×Block Size

- Each PE uses this formula to pick the correct label from the block.
    
- Both sides calculate the same label, so they agree without explicit per‑PE signaling.
    
👉 Think of it like a seating chart:

- The PE advertises a “row of seats” (label block).
    
- Each remote PE has a ticket number (VE ID).
    
- The formula tells them which seat (label) to sit in.
    
- Everyone ends up in the right seat without needing a separate ticket for each person.
## 🔹 Why It Works

- **Scalable:** One update per PE per VPLS domain, not n×PE updates.
    
- **Efficient:** Label blocks grow only when higher VE IDs appear.
    
- **Unique:** Each pseudowire still gets a unique label, but it’s derived mathematically.
    

## 🔹 Interview‑Ready Summary

> “With BGP‑only VPLS, we remove LDP sessions and let BGP handle both autodiscovery and label signaling. The challenge is that each PE needs a unique label per remote PE for MAC learning. Instead of sending multiple updates, BGP advertises one update per VPLS domain containing a block of labels. Each PE uses its VE ID and a formula to calculate which label to use from the block. This ensures uniqueness, scalability, and efficiency. The data plane remains the same — bridging, MAC learning, and split horizon — but the control plane is simplified to BGP only.”

## 🔹 Memory Map (Quick Recall)

Code

```
BGP-only VPLS
   ├─ No LDP sessions
   ├─ Each PE advertises one update per VPLS
   ├─ Update contains:
   │    ├─ VE ID (unique per PE)
   │    ├─ Block offset
   │    ├─ Block size
   │    └─ Label base
   ├─ Formula → calculates label per pseudowire
   └─ Result → unique labels, scalable, efficient
Data plane unchanged → bridging, MAC learning, split horizon
```