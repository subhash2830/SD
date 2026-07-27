# CCIE/CCDE — LDP (Label Distribution Protocol)
*Simple explanations, CCDE-level design depth, interview answers, CLI, and a concept memory map — covering all 14 LDP labs.*

---

## 1. LDP and ECMP

**What:** When multiple equal-cost IGP paths exist to a destination, LDP by default programs the LFIB with labels for **all** of them, matching the IGP's ECMP set — LDP doesn't make its own path decisions, it simply follows CEF/RIB.

**Why it matters (CCDE lens):** LDP has no independent path-selection logic — it is entirely a passive consumer of whatever the IGP/RIB decides. This is a foundational design fact: if you want to influence MPLS forwarding paths, you tune the **IGP** (metrics, ECMP knobs), not LDP itself. This is the direct contrast with SR-TE/RSVP-TE, which CAN compute and pin an explicit path independent of the IGP shortest path — a key "why would you use SR/TE over plain LDP" argument.
**Real-world example:** A P router with two equal-cost core links to the next hop load-shares labeled traffic across both automatically — the same hashing (5-tuple, label-aware) used for unlabeled ECMP applies to LSPs, giving core bandwidth efficiency for free without any LDP-specific configuration.

---

## 2. LDP and Static Routes

**What:** LDP labels only get used for a destination if the FIB's next-hop for that prefix matches an address the LDP neighbor advertised as bound to itself. A static route pointing out a **multi-access** interface (no explicit next-hop IP) creates a "glean" adjacency that never resolves to a specific LDP peer, so the LFIB entry is never built — even though the LDP session itself may be up and exchanging labels fine.

**Why it matters (CCDE lens):** This is a classic "the control plane looks fine but forwarding is broken" trap — a top real-world troubleshooting pattern. The fix is always to make the route recursively resolvable to a specific peer address: use a next-hop IP (not just an outgoing interface) on multi-access media, or use a point-to-point-like interface. On serial/POS/point-to-point interfaces this isn't an issue since there's only one possible next hop.
**Real-world example:** Running MPLS core reachability via static routes instead of an IGP (uncommon, but seen in very small/simple SP cores or lab environments) — you must always specify the next-hop IP explicitly, never just the interface, or labels silently never get programmed into the LFIB.

---

## 3. LDP Timers

**What:** Two independent timer families: **Discovery** (Hello interval/hold time, UDP-based, controls neighbor discovery and liveness) and **Session** (Keepalive interval/hold time, TCP-based, controls the LDP session itself once established).

**Why it matters (CCDE lens):** Same convergence-speed-vs-CPU/stability trade-off theme as ISIS/BFD timers — but critically, **LDP has no BFD-like hardware-offload fast-failure path of its own**. In practice, most designs rely on the IGP (with its own BFD session) to detect the physical failure and withdraw the route, which then triggers LDP label withdrawal — rather than tuning LDP's own hello/hold timers aggressively. Aggressively low LDP session timers mainly protect against a "half-open"/stuck TCP session scenario, not physical link failure, which is properly the IGP/BFD's job.
**Real-world example:** In most production SP designs, LDP hello/hold timers are left near default, and fast failure detection is delegated entirely to IGP+BFD — over-tuning LDP timers is a common junior-design mistake that adds complexity without meaningfully improving convergence.

```
mpls ldp discovery hello interval <sec> holdtime <sec>
mpls ldp holdtime <sec>          ! session keepalive hold time
```

---

## 4. LDP Authentication

**What:** LDP authentication uses **TCP MD5** (the identical mechanism to BGP's TCP-AO/MD5) to protect the LDP TCP session — it does **not** authenticate the UDP discovery Hellos.

**IOS-XE vs IOS-XR philosophy — a real CCDE-relevant platform difference:**

| | IOS-XE | IOS-XR |
|---|---|---|
| Default password scope | `fallback` password applies to all neighbors unless overridden | Global `neighbor password` applies to all unless a specific RID override is set |
| Disruptive on change? | **No** — new password only required when session next resets, unless `mpls ldp password required` is set | **Yes** — always immediately disruptive, session is torn down and re-established |
| Per-neighbor keychain | Not direct — requires an ACL matching the peer's LDP RID + a "password option" | Can apply per-RID directly |
| Password rollover | Supported — `rollover duration <min>` lets you stage a password change with a grace period before it's required | Not a native concept — a change is just immediately disruptive |

> **Why it matters (CCDE lens):** IOS-XE's password rollover is a genuine **non-disruptive, zero-outage credential-rotation design pattern** — the same operational problem that BFD/ISIS/OSPF authentication migrations solve (Section 7/Section 7 of BFD and ISIS notes) shows up again here. IOS-XR has no equivalent — a password change on XR is a planned, disruptive maintenance event by design. This is a real operational-risk difference to call out when comparing platforms for a large-scale rollout.

```
#IOS-XE
mpls ldp password fallback LDPDEFAULT
mpls ldp neighbor 1.1.1.1 password LDP12
mpls ldp password required                          ! make disruptive like XR
mpls ldp password fallback key-chain <name>          ! fallback via keychain
mpls ldp password option 1 for 1 key-chain KC34       ! per-neighbor keychain via ACL
mpls ldp rollover duration 3                          ! grace-period password change

#IOS-XR
mpls ldp
 neighbor
  password clear LDPDEFAULT
  6.6.6.6:0 password clear LDP619
```
**Verification gotcha:** On IOS-XE, `show mpls ldp neighbor detail` can show a password as **"stale"** — configured but not yet in use because the session hasn't reset. Don't assume auth is active just because it's configured; confirm the session actually re-established with the new password (`clear mpls ldp neighbor` forces it).

---

## 5. LDP Session Protection

**What:** Keeps the LDP **session and label bindings** alive across a local-link flap by falling back to a **targeted (unicast, non-multicast-Hello) session** to the neighbor's loopback, as long as an alternate IGP path still exists. A holdup timer (default 86400s/24h) bounds how long it will hold bindings if the direct link never returns.

**Why it matters (CCDE lens) — the precise distinction from LDP/IGP Sync:**
- **Session Protection** = faster LDP **re-convergence** after a flap (bindings are preserved, no LDP renegotiation needed when the link returns).
- **LDP/IGP Sync** (Section 6) = prevents **blackholing** by holding the IGP back until LDP is ready in the first place.
These solve *different* problems and are often used together, not as alternatives — a common interview confusion point.
**Mechanism:** Session protection is nothing more than an automatically-managed targeted Hello — you can achieve the identical result manually with `mpls ldp neighbor <ip> targeted` on one side and `discovery targeted-hello accept` on the other. The only functional difference is the built-in holdup timer.
```
mpls ldp session protection [duration <sec>|infinite]
mpls ldp session protection for <acl>        ! restrict to specific peer RIDs
```

---

## 6. LDP/IGP Sync (OSPF & ISIS)

**What:** Solves the inverse problem from Session Protection: if the **IGP converges before LDP finishes exchanging labels** on a newly-up link, traffic can be routed over that link before it's actually labeled — causing a real (if brief) blackhole. LDP/IGP Sync holds the IGP's advertised metric at **max value** on that link until LDP reports "sync achieved," discouraging IGP from using the link prematurely.

**Two key timers:**
- **Sync delay** (both platforms): how long to wait *after* LDP converges before restoring the real metric — protects against LDP briefly flapping right after coming up.
- **Holddown timer** (**IOS-XE only**): bounds how long the IGP will refuse to form an adjacency while waiting for LDP Hellos at all. Without it, IOS-XE can wait **indefinitely** ("DOWN (waiting for LDP)") if LDP never comes up on that link — a real outage risk if LDP is misconfigured or intentionally disabled on one side.

> **Why it matters (CCDE lens) — the single most important platform-behavior difference in this whole LDP section:** IOS-XR does **not** implement a holddown timer at all — if LDP never comes up, the IGP adjacency still forms anyway (with sync status "not ready"), just without ever getting a metric reduction. This means **on IOS-XR you cannot accidentally cause an IGP outage by disabling LDP on one side of a Sync-enabled link; on IOS-XE, you can**, unless you've explicitly set a bounded holddown timer. This is a genuine platform-choice/config-checklist item for any design mixing XE and XR with LDP/IGP sync enabled.

```
#IOS-XE
router ospf 1
 mpls ldp sync
!
router isis 1
 mpls ldp sync
int Gi2.23
 mpls ldp igp sync delay 20
mpls ldp igp sync holddown 10000        ! XE-only safety valve

#IOS-XR
router ospf 1
 mpls ldp sync
!
router isis 1
 interface Gi0/0/0/0
  address-family ipv4
   mpls ldp sync
mpls ldp
 interface GigabitEthernet0/0/0/0
  igp sync delay on-session-up 20
```
**Verification:** `show mpls ldp igp sync` — look for "sync achieved" vs "sync not achieved; peer reachable." Note the OSPF *configured* interface cost won't visibly change (still shows the normal value) — you must inspect the actual **Router LSA** to see the real advertised max-metric.

---

## 7. Local Label Allocation & Advertisement Filtering (Local Allocation, Conditional Advertisement, Inbound Filtering, Advertisement Filtering Challenge)

**The core design problem:** By default, LDP allocates a local label for **every** prefix in the RIB and advertises it to **every** neighbor — including transit /30s, loopbacks of routers that will never be a VPN/service endpoint, etc. At scale, this wastes label space and LIB/LFIB memory on every router in the domain for prefixes nobody actually needs a label for.

**Three generations of solving this, in order of operational preference:**

| Technique | How it works | Trade-off |
|---|---|---|
| **Local allocation filtering** (modern, preferred) | Router only *allocates* a local label for prefixes matching a filter (e.g., `allocate global host-routes` = only /32s), full stop — never even created for other prefixes | Cleanest — one config statement per router, no per-peer ACL bookkeeping |
| **Inbound filtering** (legacy) | Router still allocates labels for everything locally, but a receiving router filters which *remote* bindings it *accepts* from a specific neighbor via ACL | Must be applied per-peer; router still wastes local resources allocating unnecessary labels |
| **Outbound / advertisement filtering** (legacy) | Router still allocates everything, but restricts which labels it *advertises* to a peer via ACL; must explicitly disable the default advertise-all behavior | Same waste problem as inbound filtering, plus you must remember `no mpls ldp advertise-labels` (XE) / `label local advertise disable` (XR) or filtering has no effect |

> **Why it matters (CCDE lens):** This is a genuine "know the history to design correctly today" topic. Legacy inbound/outbound filtering is tedious, per-peer, and error-prone (it's easy to forget the "disable default advertisement" step and have your ACL filter silently do nothing) — modern local allocation filtering (by host-route, prefix-list, or ACL) solves the SAME operational goal (only distribute labels for prefixes that need them — typically PE loopbacks) with a single global statement per router, no per-peer bookkeeping required. In a design review, recommending legacy filtering over local-allocation filtering for a greenfield build would be a red flag.

**Conditional Label Advertisement** — a related but distinct mechanism: advertise a label for a prefix to a peer **only if** another specific prefix/condition is also reachable — used to avoid advertising a label for a route that would be useless without a companion route also being present (e.g., don't advertise the service label if the underlying transport isn't viable). Conceptually the same "don't create a false promise of reachability" pattern seen in ISIS's Conditional ATT Bit.

```
#IOS-XE — local allocation filtering (preferred)
mpls ldp label
 allocate global host-routes                  ! only /32s
 allocate global prefix-list <name>            ! or by prefix-list

#IOS-XR
mpls ldp
 label
   local
    allocate for host-routes
    allocate for <ACL>

#IOS-XE — legacy inbound filtering
access-list 1 permit 2.2.2.2
mpls ldp neighbor 2.2.2.2 labels accept 1

#IOS-XE — legacy outbound filtering (must also disable default!)
no mpls ldp advertise-labels
mpls ldp advertise-labels for <prefix-acl> to <peer-acl>

#IOS-XR — legacy outbound filtering
mpls ldp
 address-family ipv4
  label
   local
    advertise
     disable
     for <prefix-ACL> to <peer-ACL>
```

---

## 8. LDP Implicit Withdraw

**What:** Per RFC 5036, if a router re-advertises a **new** label mapping for a FEC it has already advertised to a peer, the new mapping **implicitly replaces** the old one — the peer is expected to simply overwrite its LIB entry. No explicit "Label Withdraw" message is required for this specific case; it's only used when a mapping needs to be removed with no replacement.

**Why it matters (CCDE lens):** Understanding implicit withdrawal correctly matters for **packet-loss reasoning during any local-label-allocation policy change** — e.g., changing which prefixes get local labels (Section 7), or a local label allocation range change. Because the replacement is implicit, there's a brief window where downstream routers may still be using the old label while the local LFIB has already moved to the new one, until the updated mapping fully propagates. This is the kind of subtle "why did I see a few dropped packets during a policy change I thought was clean" root cause that separates CCDE-level understanding from just knowing the CLI.
**Real-world example:** Migrating from allocating labels for all prefixes to `host-routes`-only filtering (Section 7) triggers a wave of implicit label reassignments across the domain — worth doing during a maintenance window, not as a "should be transparent" live change, precisely because of this propagation-timing behavior.

---

## 9. LDP Transport Address Troubleshooting

**What:** The LDP **TCP session** (used for the actual label exchange) is established using each router's advertised **transport address** — by default, this is the **interface address** the Hello was sourced from, NOT the router's loopback/LDP Router-ID, unless explicitly configured otherwise.

**The classic failure mode:** Two routers form LDP Hello adjacency fine (UDP discovery works over the directly connected subnet), but the **TCP session never establishes** — because each router is trying to reach the other's *interface* address as the transport address, and that interface address isn't actually reachable/routed the way the loopback-based design assumes (common when using targeted sessions, or when the physical link subnet itself isn't advertised into the IGP due to prefix suppression — direct callback to ISIS Section 4.1).

**Why it matters (CCDE lens):** This is a top real-world LDP troubleshooting pattern precisely because the symptom is confusing: `show mpls ldp discovery` looks completely healthy (Hellos are fine, that's UDP/link-local), but `show mpls ldp neighbor` shows the session stuck in a non-operational state — the failure is one layer deeper than where people instinctively look first. The standard fix, and standard **design practice** for stability, is to explicitly source the transport address from the loopback:
```
#IOS-XE
mpls ldp router-id Loopback0 force
int Gi2.23
 mpls ldp discovery transport-address interface   ! or explicitly set loopback

#IOS-XR
mpls ldp
 router-id 2.2.2.2
 interface GigabitEthernet0/0/0/0
  transport-address 2.2.2.2
```
> **Design best practice:** Standardize LDP transport address on the loopback domain-wide from day one — it makes LDP TCP sessions resilient to any single physical link's reachability/suppression status and is consistent with how virtually every other control-plane session (BGP, targeted LDP, TE) is designed to peer via loopback in a mature SP core.

---

## 10. LDP Static Labels

**What:** Manually binds a specific local label value to a specific FEC/prefix, bypassing LDP's normal dynamic allocation for that entry — and can also manually inject a *remote* binding for a peer that isn't actually running LDP.

**Why it matters (CCDE lens):** Two legitimate design uses: (1) **interop with a non-LDP-speaking device** at a domain boundary where you still need a consistent, predictable label to hand off traffic — a static/manual label stitching point; (2) **deterministic lab/test scenarios** where you need to guarantee a specific label value for validation or documentation purposes. This is the LDP-world equivalent of a manually-configured (non-signaled) pseudowire — trading protocol automation for explicit operator control, appropriate only at a small number of well-understood boundary points, not as a general design pattern (it doesn't scale and doesn't self-heal on topology change the way dynamic LDP does).

---

## 11. CCDE-Style Interview Q&A

**Q1. Why can't you influence which of several equal-cost core links carries labeled traffic by configuring LDP directly?**
LDP has no independent path-selection logic of its own — it purely reflects whatever ECMP set the IGP/RIB has already decided. To influence the actual forwarding path you must tune the IGP (metrics) or move to SR-TE/RSVP-TE, which compute paths independently of the IGP shortest path.

**Q2. A static route to a next-hop over an Ethernet interface isn't getting a label programmed into the LFIB, even though the LDP session is up. Why?**
The static route likely points only to the outgoing interface, not an explicit next-hop IP, creating a "glean" adjacency on the multi-access segment that never resolves to a specific LDP peer's bound address — LDP requires the FIB next-hop to match one of the peer's advertised addresses. Fix: specify the next-hop IP explicitly.

**Q3. What's the practical difference between LDP Session Protection and LDP/IGP Sync, and why would you use both?**
Session Protection speeds up LDP's own re-convergence after a link flap by preserving bindings via a fallback targeted session. LDP/IGP Sync prevents the IGP from routing traffic over a link before LDP has even finished converging in the first place — a blackhole-prevention mechanism, not a re-convergence-speed mechanism. They solve different failure windows and are commonly deployed together.

**Q4. Why is IOS-XR's LDP/IGP Sync arguably safer by default than IOS-XE's?**
IOS-XR has no holddown timer — if LDP never comes up on a sync-enabled link, the IGP adjacency still forms anyway (just without a metric reduction). IOS-XE, by contrast, can wait indefinitely for LDP Hellos before forming the IGP adjacency at all unless you explicitly configure a bounded holddown timer — meaning a misconfigured or disabled LDP session can cause a genuine IGP-level outage on IOS-XE that can't happen the same way on IOS-XR.

**Q5. Why is local label allocation filtering considered better design practice than legacy inbound/outbound label filtering?**
Legacy filtering still allocates labels for every prefix locally and only controls what's accepted/advertised per-peer via ACLs — tedious, per-neighbor, and easy to misconfigure (forgetting to disable default advertisement makes an outbound filter silently ineffective). Local allocation filtering (e.g., host-routes only) stops the router from ever allocating unnecessary labels in the first place, with one global statement — cleaner and less error-prone at scale.

**Q6. LDP Hellos are healthy between two neighbors but the LDP session never establishes. What's your first troubleshooting hypothesis?**
Check the transport address each side is using for the TCP session — by default it's the interface address the Hello was sourced from, not the loopback. If that interface address isn't properly reachable (e.g., due to routing policy, prefix suppression, or the design assuming loopback-based peering), the TCP session will never come up even though UDP discovery looks completely healthy.

**Q7. Why does LDP's "implicit withdraw" behavior matter when changing a label allocation policy on a live router?**
Because a new label mapping silently replaces the old one without an explicit withdraw message, there's a propagation window where downstream routers may still be using the old label while the local LFIB has already switched — a source of brief, hard-to-diagnose packet loss during what looks like a "clean" policy change. This is why such changes belong in a maintenance window.

---

## 12. Memory Map

```
LDP Core
│
├── Path Selection (Section 1)
│     LDP has NO independent path logic — pure follower of IGP/RIB
│     └─ contrasts directly with SR-TE/RSVP-TE (independent path compute)
│
├── Label-to-Route Binding (Section 2)
│     Requires FIB next-hop to match an LDP peer's bound address
│     └─ breaks silently with interface-only static routes on multi-access media
│
├── Session Lifecycle & Resilience
│     Timers (3) ── Discovery (UDP) vs Session (TCP) — usually left near
│     │              default; fast-failure detection delegated to IGP+BFD
│     Session Protection (5) ── re-CONVERGENCE speed after a flap
│     LDP/IGP Sync (6) ── prevents BLACKHOLING before LDP is ready
│     │     └─ these two are COMPLEMENTARY, not alternatives
│     └─ Transport Address (9) ── WHY the TCP session specifically may
│           never establish even when Hellos (UDP) look fine —
│           standardize on loopback to avoid this class of failure
│
├── Security (4)
│     TCP MD5, same mechanism as BGP auth
│     XE: non-disruptive by default + password rollover (safe migration)
│     XR: always disruptive on change (same "how do you change live
│          config safely" theme seen in ISIS/BFD authentication)
│
├── Label Scale Control (7)
│     Local allocation filtering (modern) supersedes
│     inbound/outbound ACL filtering (legacy) — same GOAL (only PE
│     loopbacks get labels) achieved with far less per-peer config
│     Conditional Advertisement ── same "don't falsely promise
│     reachability" pattern as ISIS Conditional ATT Bit
│     └─ Implicit Withdraw (8) ── the propagation-timing side-effect
│           of CHANGING a Section 7 policy live
│
└── Manual Override (10)
      Static Labels ── deliberate escape hatch for non-LDP boundary
      interop or deterministic testing; doesn't scale as a general pattern
```

**Recurring CCDE theme in LDP, same as ISIS/BFD:** *what happens during the transition/change window* — implicit withdraw during a filtering-policy change, session protection during a link flap, IGP sync during initial convergence, password rollover during a credential change. LDP's "hard" interview questions are almost always about these transition windows, not steady-state behavior.

---

## 13. CLI Cheat Sheet

| Purpose | Command |
|---|---|
| Enable LDP via IGP autoconfig | `mpls ldp autoconfig` (under OSPF/ISIS, XE) / `mpls ldp auto-config` (XR) |
| Discovery/session timers | `mpls ldp discovery hello interval <s> holdtime <s>` / `mpls ldp holdtime <s>` |
| Session protection | `mpls ldp session protection [duration <s>\|infinite]` |
| Targeted session (manual) | `mpls ldp neighbor <ip> targeted` / `discovery targeted-hello accept` |
| LDP/IGP sync (under IGP process) | `mpls ldp sync` |
| LDP/IGP sync delay (per-interface) | `mpls ldp igp sync delay <s>` (XE) / `igp sync delay on-session-up <s>` (XR) |
| LDP/IGP sync holddown (XE only) | `mpls ldp igp sync holddown <ms>` |
| Fallback/default password (XE) | `mpls ldp password fallback <pwd>` |
| Per-neighbor password (XE) | `mpls ldp neighbor <ip> password <pwd>` |
| Make password change disruptive (XE) | `mpls ldp password required` |
| Password rollover (XE) | `mpls ldp rollover duration <min>` |
| Global/per-RID password (XR) | `mpls ldp neighbor password clear <pwd>` / `<rid>:0 password clear <pwd>` |
| Local allocation filtering (XE) | `mpls ldp label allocate global host-routes \| prefix-list <name>` |
| Local allocation filtering (XR) | `mpls ldp label local allocate for host-routes \| <ACL>` |
| Legacy inbound filtering | `mpls ldp neighbor <ip> labels accept <acl>` |
| Legacy outbound filtering (XE) | `no mpls ldp advertise-labels` + `mpls ldp advertise-labels for <acl> to <acl>` |
| Legacy outbound filtering (XR) | `label local advertise disable` + `advertise ... for <ACL> to <ACL>` |
| Set LDP router-id / transport address | `mpls ldp router-id Loopback0 force` (XE) / `router-id <ip>` + `transport-address <ip>` (XR) |
| Verify discovery (UDP) | `show mpls ldp discovery` |
| Verify session (TCP) state, auth | `show mpls ldp neighbor [detail]` |
| Verify IGP sync status | `show mpls ldp igp sync` |
| Verify LFIB contents | `show mpls forwarding-table` |

---
*Source: CCIE-SP v5.1 Labs — LDP section (14 labs): LDP and ECMP, LDP and Static Routes, LDP Timers, LDP Authentication, LDP Session Protection, LDP/IGP Sync (OSPF), LDP/IGP Sync (ISIS), LDP Local Allocation Filtering, LDP Conditional Label Advertisement, LDP Inbound Label Advertisement Filtering, LDP Label Advertisement Filtering Challenge, LDP Implicit Withdraw, LDP Transport Address Troubleshooting, LDP Static Labels. Some sub-topics supplemented with standard RFC 5036 LDP behavior where the specific lab page content was not directly retrievable.*
