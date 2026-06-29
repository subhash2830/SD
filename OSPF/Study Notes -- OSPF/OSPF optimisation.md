[ Tunnig ospf performance](https://ine.com/blog/2009-12-31-tuning-ospf-performance)


# OSPF Tuning: Convergence & Scalability — Interview Notes

> **Author's Note:** These notes are based on practical 30-year network infrastructure experience. Written for interview preparation and quick revision.

---

## 🧠 The Core Tension — Remember This First

> **"Faster convergence = Less stability. More stability = Slower convergence."**

These two always fight each other. Your job as a network engineer is to **find the right balance** for your environment.

**Key Definitions:**

- **Convergence** — The process of restoring a stable view of the network after a change
- **Scalability** — The routing protocol's ability to remain stable and well-behaving as the network grows

---

## 1️⃣ Incremental SPF (iSPF)

### What Problem Does It Solve?

Classic Dijkstra SPF recalculates the **entire** Shortest Path Tree (SPT) every time _anything_ changes — even a tiny stub link going up/down. That's wasteful.

- Classic SPF complexity: **O(N log N)** where N = number of nodes in the area
- Modern routers take tens to hundreds of milliseconds for a full SPF run
- iSPF was developed as early as **1980** (by Eric Rosen!)

### The Big Idea

**Save the SPT in memory** after the first run, then use it smartly for future changes instead of rebuilding from scratch.

### 3 Optimization Properties

|Property|Scenario|What Happens|
|---|---|---|
|**1 — Leaf Node Change**|New router or stub link added/removed|Just "extend" the existing tree. Simple distance-vector-like computation. No full SPF needed ✅|
|**2 — Link Fails but NOT in SPT**|e.g., a backup/redundant link fails|Zero SPF calculation needed ✅|
|**3 — Transit Link Fails IN the SPT**|e.g., core link goes down|Only recalculate paths **downstream** of failure, not the whole tree ✅|

### Key Interview Points

- **Property 1** is the most impactful in real networks
- iSPF works **best in sparse topologies** (hub-spoke, ring)
- In a **fully-meshed topology**, iSPF gives **zero benefit** — must recalculate everything
- The **further from root** the failure is → fewer nodes to recalculate → faster iSPF run
- Different routers have different SPTs — the same failure may have different iSPF impact per router
- Uses **slightly more memory** (~2×N, where N = nodes in area) — cost of storing the tree

### iSPF vs IS-IS PRC — Know the Difference

||IS-IS PRC|OSPF iSPF|
|---|---|---|
|**Mechanism**|Separates topology (IS reachability) from prefix info using TLVs in LSPs|Saves the SPT and reuses it via the 3 properties|
|**Stub link change**|Never triggers full SPF — PRC handles it|iSPF Property 1 handles it equivalently|
|**Transit link change**|Always triggers full SPF|iSPF Property 3 limits scope|
|**Historical advantage**|IS-IS was historically more scalable for single-area designs|iSPF closed the gap|

> **Old OSPF problem:** Type-1 LSA bundled both topology AND prefix info — any change triggered full SPF. iSPF (Property 1) fixed this, making OSPF equivalent to IS-IS PRC. Today both protocols are equally efficient.

### How to Enable

```
router ospf 1
 ispf
```

> Disabled by default. Enabling it uses slightly more memory proportional to 2×N nodes.

---

## 2️⃣ LSA Group Pacing

### What Problem Does It Solve?

OSPF LSAs have a **30-minute refresh cycle** and a **10-minute checksum/aging cycle**. Without tuning:

- All LSAs refresh at the same time → **massive CPU spikes + flooding bursts** every 30 minutes
- Classic **synchronization problem**

### Three Approaches — Evolution of the Solution

|Approach|Method|Problem|
|---|---|---|
|**Original (Bad)**|Refresh ALL LSAs every 30 min simultaneously|Huge CPU and flooding burst every 30 min|
|**Individual Aging**|Each LSA has its own independent timer|Many tiny floods scattered throughout — fragmentation problem|
|**Group Pacing ✅**|Group LSAs with similar age, refresh together in batches|Controlled, balanced bursts — best of both worlds|

### How Group Pacing Works

- Default pacing interval = **240 seconds**
- Router groups LSAs close to their half-life (30 min) and refreshes them **together as a batch**
- Same grouping applied to **checksumming** and **aging**
- Shorter interval = smaller, more frequent bursts
- Longer interval = larger, less frequent bursts

### The 3 Pacing Timers

```
timers pacing ?
  flood           → Controls interface flood list (default: 33ms)
  lsa-group       → Controls refresh/age/checksum grouping (default: 240s)
  retransmission  → Groups unacknowledged LSA retransmissions (default: 66ms)
```

**What each timer does:**

- **`lsa-group`** — Groups LSAs for refreshing, aging, and checksumming. For very large LSDBs, reduce this (inversely proportional to DB size) to avoid large surges
- **`retransmission`** — Groups unacknowledged LSAs waiting to be retransmitted. Allows better packing of LSA info in IP packets. Measured in milliseconds
- **`flood`** — Groups LSAs on the interface flood list. Instead of flooding each LSA immediately, waits the pacing interval to pack more into a single update packet. Measured in milliseconds

### Key Interview Points

- For **very large LSDBs** → reduce `lsa-group` timer (inversely proportional to DB size)
- Group pacing improves **performance and scalability** but does **NOT speed up convergence**
- The flood and retransmission timers are in **milliseconds** because they are real-time operations
- Default values are generally optimal for most networks

### Verify Current Settings

```
show ip ospf | inc transmission|pacing
 LSA group pacing timer 240 secs
 Interface flood pacing timer 33 msecs
 Retransmission pacing timer 66 msecs
```

---

## 3️⃣ SPF & LSA Generation Throttling

### What Problem Does It Solve?

A flapping link can trigger **hundreds of SPF runs** and **LSA floods** per minute — destroying CPU and destabilizing the network. Throttling puts a **speed limiter** on OSPF's reactions.

### Throttling vs Dampening — Know the Difference

||Throttling|Dampening|
|---|---|---|
|**What it does**|Slows down responses, increases wait time|Suppresses events entirely|
|**Effect**|Filters high-frequency noise|Hides events from the protocol|
|**Protocol still aware?**|Yes, just slower to react|No, events are hidden|

### The Exponential Back-off Mechanism

```
timers throttle spf <start> <increment> <max_wait>
timers throttle lsa <start> <increment> <max_wait>
```

**Three parameters:**

- **`start`** — Initial wait time (ms) before first SPF after an event
- **`increment`** — Base interval for next hold window
- **`max_wait`** — Maximum hold time ceiling

### How the Timer Works — Step by Step

```
Network was stable → Event occurs (e.g., LSA arrives or link flaps)
  ↓
Wait "start" ms before running SPF
  ↓
Next hold-time = "increment" ms
  ↓
Event occurs during hold? → Double the next wait (2× increment)
  ↓
Event again? → Double again (4× increment)
  ↓
Keep doubling until "max_wait" is reached
  ↓
Stay at max_wait as long as events keep occurring
  ↓
No events for 2× max_wait? → Reset back to "start"
```

> **In plain English:** "The more the network freaks out, the longer OSPF waits before reacting. Once things calm down for long enough, it goes back to reacting quickly."

### Key Interview Points

- Both SPF and LSA throttling are **ON by default** — don't disable them
- Only **reduce** these timers if you genuinely need faster convergence (e.g., voice/video/financial networks)
- Reducing timers = faster convergence but **less stability**
- Same exponential logic applies to both:
    - **SPF throttling** — reaction to incoming LSAs
    - **LSA generation throttling** — reaction to local link events
- After `2 × max_wait` with no events, the timer **resets to start** — network assumed stable

---

## 🎯 Quick Summary Table — For Interviews

|Feature|Problem Solved|Affects Convergence?|Affects Scalability?|Default State|
|---|---|---|---|---|
|**iSPF**|Full SPF on every change|✅ Yes (faster)|✅ Yes|❌ Disabled|
|**LSA Group Pacing**|CPU bursts from LSA refreshes|❌ No|✅ Yes|✅ Enabled|
|**SPF/LSA Throttling**|CPU waste from link flaps|✅ Yes (slower, intentionally)|✅ Yes|✅ Enabled|

---

## 💬 One-Liner Answers for Interviews

**"What is iSPF?"**

> "It saves the shortest path tree and only recomputes the parts affected by a change, instead of running full Dijkstra every time."

**"What is LSA group pacing?"**

> "It groups LSAs with similar ages together for refreshing and checksumming to avoid CPU spikes — like batching tasks instead of doing them one by one."

**"What is SPF throttling?"**

> "It uses an exponential back-off timer so that during link flaps, OSPF slows down its reactions instead of burning CPU on every flap."

**"Why not always use the fastest convergence settings?"**

> "Because fast convergence makes the protocol more sensitive to instability. You get faster recovery but higher risk of routing loops and CPU exhaustion during flaps."

**"When would you disable or reduce throttling?"**

> "Only in environments where sub-second convergence is mandatory — like VoIP, financial trading, or video conferencing networks — and only after thorough stability testing."

**"What's the difference between OSPF iSPF Property 1 and IS-IS PRC?"**

> "Functionally the same — both avoid full SPF when only stub links or prefixes change. IS-IS achieves this through TLV separation in LSPs; OSPF achieves it by saving the SPT and applying distance-vector logic for leaf node changes."

---

## 🏗️ Design Recommendations (From Experience)

1. **Enable iSPF** in any area with more than 20–30 routers — the memory cost is negligible, the CPU savings are real
2. **Leave throttling defaults** unless you have a specific, tested reason to change them
3. **Reduce `lsa-group` pacing** in large networks (500+ LSAs) to avoid flooding surges
4. **Never tune convergence in isolation** — always test in a lab with realistic traffic loads
5. **Pair these features with good design** — area partitioning, route summarization, and dampening remain your best scalability tools

---

## 📚 Related Topics to Study

- OSPF Area Design (Backbone, Stub, NSSA, Totally Stub)
- Route Summarization and its impact on LSA flooding
- BFD (Bidirectional Forwarding Detection) for fast failure detection
- IS-IS vs OSPF scalability comparison
- OSPF Fast Hello (sub-second hellos) and its tradeoffs
- RFC 1245 — SPF complexity analysis
- OSPF NSR (Non-Stop Routing) and GR (Graceful Restart)



