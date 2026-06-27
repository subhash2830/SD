---
uid:
title: ISIS
aliases:
topic: Timer
date:
tags:
  - protocol/isis
status:
priority:
---
# IS-IS Hello & Hold Timers (Architect Notes + Interview)  

  

## 1. Hello Interval  

  

- What: Time between IS-IS Hello (IIH) packets used to maintain neighbor adjacency.  

- Why: Ensures continuous neighbor liveliness detection; without it, adjacency state becomes stale.  

- How: Default = 10 sec (non-DIS), DIS sends Hellos faster (~Hello/3).  

- Risk: Too high → slow failure detection; too low → CPU load and frequent flaps.  

- Example: WAN link failure detected after 30s (default hold) → application timeout complaints.  

- Takeaway: Hello interval defines detection speed baseline but should not be aggressively reduced alone.  

  

👉 Interview Angle:  

Hello interval controls how frequently neighbors confirm liveliness; tuning it directly impacts convergence speed vs network stability.  

  

---  

  

## 2. Hold Interval (Dead Timer)  

  

- What: Time a router waits before declaring neighbor down (default = 3 × Hello).  

- Why: Prevents false neighbor drops due to temporary packet loss.  

- How: Controlled via multiplier:  

isis hello-multiplier

```
- Risk: Too low → false flaps; too high → slow convergence.
- Example: Packet drops on congested link cause adjacency reset when hold timer too aggressive.
- Takeaway: Hold timer must balance reliability and failure detection sensitivity.

👉 Interview Angle:
Hold timer defines failure detection threshold and is directly tied to hello interval; improper tuning leads to either slow convergence or instability.

---

## 3. DIS Hello Behavior

- What: DIS sends Hello packets at faster rate (Hello / 3).
- Why: DIS maintains LAN LSDB sync (CSNP responsibility), so faster detection is required.
- How: Automatically derived from standard hello interval.
- Risk: In large VLANs, DIS may face higher control-plane load.
- Example: High-density metro VLAN → DIS overloaded due to fast hello + CSNP operations.
- Takeaway: DIS plays critical control-plane role; faster hello ensures LAN stability.

👉 Interview Angle:
DIS uses faster hellos because it is responsible for database synchronization, so it must detect failures more quickly than other routers.

---

## 4. Sub-Second Hello (Fast Convergence)

- What: Enables millisecond-level hello timers.
- Why: Required in modern DC and low-latency environments.
- How:
```

isis hello-interval minimal isis hello-multiplier 4

```
→ Hello ≈ 250 ms, Hold ≈ 1 sec
- Risk:
- High CPU usage
- Increased risk of false neighbor flaps
- Example: Data center fabric achieves sub-second convergence but needs stable links.
- Takeaway: Use only on high-quality links; otherwise combine with BFD.

---

## 5. Design Problem & Solution (STAR Applied)

- S: Default IS-IS timers (10s/30s) caused slow convergence during link failures in critical application network.
- T: Improve failure detection time without introducing instability.
- A: Instead of aggressively reducing hello timers, implemented BFD with moderate hello interval.
- R: Achieved sub-second failover with stable adjacency and low CPU overhead.

Short Explanation:
Hello timers alone are not ideal for fast convergence; combining moderate timers with BFD gives optimal results.

- Final Line:
In modern design, hello timers provide baseline detection, while BFD delivers fast and reliable convergence.

---

## 6. Design Guidelines (Architect View)

- Use default or moderate timers for stability
- Use BFD for fast convergence (preferred over aggressive hello tuning)
- Maintain consistent timers across network
- Be cautious with fast timers in large broadcast domains
- Always consider CPU impact before tuning

---

## 7. Quick Recall (Interview)

- Default Hello: 10 sec  
- Default Hold: 30 sec  
- DIS Hello: Hello / 3  
- Fast Hello: ~250 ms  
- Hold = Hello × multiplier  
- Best Practice: Use BFD instead of aggressive timer tuning

---

## 8. Final Architect Insight

Hello and hold timers are foundational for adjacency management, but in modern networks, 
they are not the primary convergence mechanism—designs rely on BFD for speed and use 
timers mainly for stability and backup detection.
```

---

# ✅ ✅ What You Achieved with This Format

- ✅ Concept clarity (What/Why/How)
- ✅ Real-world thinking (risks + examples)
- ✅ Interview-ready explanations inline
- ✅ STAR used **only where relevant (not forced)**
- ✅ Compact, readable, reusable

---

If you want next level 🚀: ✅ I can convert your entire IS-IS notes into this format  
✅ Or create **“Top 30 Interview Concepts (same style)”**  
✅ Or simulate **mock interview Q&A from these notes**