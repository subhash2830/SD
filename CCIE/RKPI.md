# BGP RPKI — CCDE Notes

## 1. Subtopics

### 1.1 RPKI Fundamentals — ROAs and the Trust Chain
**What:** Resource Public Key Infrastructure (RPKI) is a cryptographic framework where a Regional Internet Registry (RIR, e.g., ARIN/RIPE) signs certificates attesting which AS is authorized to originate which prefix, encoded as Route Origin Authorizations (ROAs) — each ROA states (prefix, max-length, origin-AS).

**Why it matters (CCDE lens):** RPKI validates *origin*, not the full AS-path — a CCDE-level misunderstanding is assuming RPKI stops all route hijacks; it only stops origin mismatches (someone else originating your prefix), not path-based attacks like AS-path prepend spoofing further along the path (that's what BGPsec/ASPA attempt to address, and adoption there is far lower). Design conversations must be precise about what threat model RPKI actually covers.

**Real-world example:** In 2022, several notable real-world route leaks/hijacks (origin mismatches) were caught and mitigated in near-real-time by networks that had already deployed strict RPKI-based origin validation drop policies — this is the canonical justification CCDE candidates should cite for ROV adoption urgency.

**CLI (viewing, no direct CLI creates ROAs — done at RIR):**
```
show bgp rpki table
```

### 1.2 RTR Protocol (Router-to-Cache Communication)
**What:** The RPKI-to-Router (RTR) protocol (RFC 6810/8210) is how a router pulls validated ROA data from a local RPKI validator/cache (e.g., Routinator, RIPE Validator, Cloudflare's rpki-client output) — routers themselves never talk to the RIR/global RPKI repositories directly.

**Why it matters (CCDE lens):** The validator cache is now a critical control-plane dependency — if it becomes unreachable or stale, routers fall back to their configured behavior (typically last-known-good state, but this varies by platform/timeout config). CCDE scenarios test whether you've designed cache redundancy (multiple RTR sessions to independent validator instances) and understand the failure mode if all caches are unreachable simultaneously (state persists until expiry, then treated as unknown — NOT automatically invalid).

**Real-world example:** A provider running a single Routinator instance for RTR suffers a validator VM crash; because they hadn't configured a second RTR session to a backup validator, RPKI state ages out after the refresh interval and all prefixes silently revert to "NotFound" — invalid-drop policies stop dropping anything, quietly disabling the protection during an active incident window.

**CLI:**
```
router bgp 65000
 rpki server 10.0.0.100
  transport tcp port 8282
 rpki server 10.0.0.101
  transport tcp port 8282
```

### 1.3 Validation States — Valid / Invalid / NotFound
**What:** Each received BGP route is checked against the ROA table and classified: Valid (origin-AS and prefix-length match a ROA), Invalid (a covering ROA exists but origin-AS or max-length doesn't match), or NotFound/Unknown (no covering ROA exists at all).

**Why it matters (CCDE lens):** NotFound does NOT mean safe — it means "no attestation exists," which is the default state for the majority of the internet's prefixes today given incomplete ROA adoption. A CCDE design mistake is treating RPKI as binary (valid=good, everything else=bad) — the actual policy decision is almost always "drop/depref Invalid, treat NotFound as normal/unprotected," because dropping NotFound routes wholesale would black-hole huge swaths of legitimate, simply-unsigned internet routes.

**Real-world example:** A network sets local-preference lower for Invalid routes and denies them outright at the edge, while leaving NotFound routes to compete normally on other BGP attributes — this is the standard, safe rollout policy recommended by virtually every major operator's RPKI deployment guide.

**CLI:**
```
route-map RPKI-POLICY permit 10
 match rpki invalid
 set local-preference 50
route-map RPKI-POLICY permit 20
 match rpki valid
 set local-preference 200
route-map RPKI-POLICY permit 30
!--- notfound falls through unmodified
```

### 1.4 Invalid-Drop vs Depref Policy Design
**What:** The two dominant enforcement postures — outright reject (deny) Invalid routes at ingress, versus merely depreferencing them (lower local-preference) so they only lose to a Valid/NotFound alternate path but aren't unreachable if no alternative exists.

**Why it matters (CCDE lens):** This is a real availability-vs-security tradeoff CCDE loves to probe: hard-drop is the "correct" security posture per industry best practice (MANRS), but it means a single-homed customer whose upstream mistakenly created a bad ROA (max-length too restrictive, wrong origin) becomes completely unreachable rather than just deprioritized. CCDE expects candidates to recommend hard-drop for peering/transit edges (where alternates typically exist) but caution around drop-only policies where the Invalid route might be a sole path.

**Real-world example:** An operator hard-drops Invalid routes at all transit/peering edges but discovers a small single-homed customer's prefix went Invalid due to the customer's own ROA misconfiguration (wrong max-length) — with drop policy, the customer is now globally unreachable, illustrating why ROA hygiene on the origin side matters as much as validator policy on the receiving side.

**CLI:**
```
route-map RPKI-STRICT permit 10
 match rpki valid
route-map RPKI-STRICT permit 20
 match rpki notfound
route-map RPKI-STRICT deny 30
 match rpki invalid
```

### 1.5 Max-Length Pitfall in ROA Design
**What:** A ROA's max-length field defines the longest prefix length permitted to be originated for that address block by that AS — if a network later needs to announce a more-specific (e.g., for traffic engineering or DDoS mitigation) beyond the ROA's max-length, that more-specific becomes Invalid.

**Why it matters (CCDE lens):** This is one of the most common real-world self-inflicted RPKI outages — an operator creates a ROA for /24 with max-length /24 (no more-specifics allowed), then later needs to announce a /25 during a DDoS mitigation or TE event, and that /25 gets dropped by every network enforcing invalid-drop policy globally. CCDE design must explicitly plan max-length to accommodate legitimate future de-aggregation, not just the current announcement.

**Real-world example:** A network creates ROAs matching exactly their current announcements with no max-length headroom; six months later during a DDoS event they de-aggregate to /26s for scrubbing-center redirection, and those /26s are Invalid everywhere RPKI enforcement is strict, worsening the outage instead of mitigating it.

**CLI (conceptual ROA definition at RIR portal, not router CLI):**
```
ROA: prefix 203.0.113.0/24, max-length /24, origin AS65000   <- too strict
ROA: prefix 203.0.113.0/24, max-length /26, origin AS65000   <- allows future de-aggregation
```

### 1.6 RPKI on IOS-XE vs IOS-XR — Enabling and Validation
**What:** Both platforms support RTR-based origin validation but differ in configuration syntax and default behavior for unvalidated/expired cache state; IOS-XR additionally exposes more granular per-neighbor RPKI-based policy knobs in some releases.

**Why it matters (CCDE lens):** Platform inconsistency across a mixed-vendor or mixed-OS estate is a real operational risk — CCDE won't test raw syntax memorization but will test whether you've accounted for behavioral differences (e.g., stale-cache timers, default bestpath treatment of Invalid before policy is applied) when designing a consistent RPKI posture across a heterogeneous network.

**Real-world example:** A provider running both IOS-XE and IOS-XR PEs at different RPKI-validator refresh/expiry defaults inadvertently has inconsistent invalid-route treatment during a validator outage — XE boxes treat aged-out state one way, XR boxes another — creating asymmetric routing during the incident.

**CLI (IOS-XE):**
```
router bgp 65000
 bgp rpki server tcp 10.0.0.100 port 8282
```
**CLI (IOS-XR):**
```
router bgp 65000
 rpki server 10.0.0.100
  transport tcp port 8282
  refresh-time 300
```

### 1.7 Beyond Origin Validation — ASPA and BGPsec (Awareness Level)
**What:** ASPA (Autonomous System Provider Authorization) and BGPsec are newer/emerging mechanisms addressing path validation (not just origin) — ASPA validates that an AS-path's adjacent AS-hops reflect legitimate provider relationships; BGPsec cryptographically signs the entire AS-path.

**Why it matters (CCDE lens):** CCDE expects awareness that ROV (origin validation) is table-stakes but does NOT prevent an attacker from originating correctly but forging a false upstream AS-path (route leak spoofing legitimate origin). Candidates should be able to articulate that ASPA is the more deployment-realistic near-term path-validation mechanism (lower operational/crypto overhead than BGPsec, which has seen minimal real-world adoption due to per-hop signing performance and incremental-deployment challenges).

**Real-world example:** A route leak where the correct origin AS is preserved but an unauthorized transit AS inserts itself mid-path would pass RPKI ROV cleanly (origin is valid) but could be caught by ASPA if deployed — illustrating why ROV alone is an incomplete security story that CCDE candidates must be able to explain accurately rather than overstating RPKI's coverage.

---

## 2. Interview Q&A

**Q1: What specifically does RPKI Origin Validation protect against, and what does it explicitly NOT protect against?**
A: It protects against origin mismatches — an unauthorized AS originating a prefix it has no ROA for. It does NOT validate the rest of the AS-path, so path-based attacks (unauthorized transit AS insertion, route leaks with correct origin) pass ROV cleanly; that gap is what ASPA/BGPsec attempt to close.

**Q2: Why is "NotFound" not treated the same as "Invalid" in almost every production RPKI policy?**
A: NotFound just means no ROA exists for that prefix — given incomplete global ROA adoption, treating NotFound as unsafe would black-hole a large fraction of otherwise-legitimate, simply-unsigned internet routes. Standard practice is to drop/depref only Invalid, and let NotFound compete normally.

**Q3: What happens to RPKI-based route treatment if all RTR sessions to validator caches go down simultaneously?**
A: Routers retain last-known-good ROA state until the configured refresh/expiry timers lapse; after that, behavior reverts toward treating prefixes as unvalidated (NotFound-like), which — critically — means invalid-drop policies stop enforcing anything during the outage window unless validator redundancy was designed in.

**Q4: Describe the max-length ROA pitfall and how you'd design around it.**
A: Setting max-length equal to the current announcement's prefix length blocks any future legitimate de-aggregation (e.g., DDoS scrubbing more-specifics or TE) — those more-specifics become globally Invalid under strict enforcement. Design ROAs with enough max-length headroom to cover realistic future de-aggregation scenarios, balanced against not being so permissive it enables hijack of arbitrary sub-prefixes.

**Q5: When would you choose depref-only over hard-drop for Invalid routes, and what's the tradeoff?**
A: Depref-only is safer at points in the network where an Invalid route might be the sole path to a destination (e.g., limited peering diversity) — it avoids total unreachability from a misconfigured ROA at the risk of still accepting a genuinely malicious Invalid route if no better alternative exists. Hard-drop is preferred at transit/peering edges with route diversity, per MANRS best practice.

**Q6: Why can heterogeneous IOS-XE/IOS-XR RPKI configuration create asymmetric routing during a validator outage?**
A: The two platforms can have different default behaviors and timers for handling aged-out/stale RPKI cache state — if refresh/expiry defaults differ, one platform's routers might treat a route as still-valid past the outage window while the other reverts to unvalidated treatment, causing inconsistent route acceptance/preference across the network for the same prefix.

**Q7: Why is ASPA generally considered more deployment-realistic than full BGPsec in the near term?**
A: BGPsec requires cryptographic signing of every AS-hop in the path in real time, adding meaningful per-hop performance/operational overhead and requiring near-universal adoption to be effective — ASPA instead validates provider/customer relationship legitimacy at a coarser granularity with far lower operational cost, making incremental deployment realistic.

**Q8: A customer's prefix suddenly becomes globally unreachable after they announce a more-specific for DDoS mitigation. What's the most likely RPKI-related cause, and how do you fix it going forward?**
A: Most likely cause: their ROA's max-length doesn't cover the more-specific prefix length they just announced, so it's Invalid everywhere invalid-drop is enforced. Fix: update the ROA to include sufficient max-length headroom before relying on more-specific announcements for future mitigation events — and audit this proactively rather than discovering it during an active incident.

---

## 3. Memory Map

```
BGP RPKI
├── Trust Chain
│    └── RIR-signed ROA (prefix, max-length, origin-AS)
├── Router ↔ Validator Communication
│    └── RTR Protocol (RFC 6810/8210)
│         ├── needs → validator redundancy (single cache = SPOF)
│         └── failure mode → stale state persists until expiry, then reverts to unvalidated (NOT auto-invalid)
├── Validation States
│    ├── Valid    → origin + prefix-length match ROA
│    ├── Invalid  → covering ROA exists, mismatch on origin or max-length
│    └── NotFound → no ROA exists at all (majority of internet today — treat as normal, NOT unsafe)
├── Enforcement Policy
│    ├── Hard-Drop Invalid   → best practice at transit/peering edges with path diversity
│    └── Depref Invalid      → safer where Invalid might be the only path (avoids total blackhole)
├── Common Design Pitfall
│    └── Max-Length too strict → blocks legitimate future de-aggregation (DDoS/TE) → self-inflicted outage
├── Platform Consistency
│    └── IOS-XE vs IOS-XR timer/default differences → asymmetric behavior during validator outage
└── Beyond Origin Validation (awareness-level)
     ├── ASPA    → validates AS-path adjacency/provider relationships, lower overhead, more deployable
     └── BGPsec  → full cryptographic AS-path signing, high overhead, minimal real-world adoption
          └── Key insight: ROV alone does NOT stop correct-origin route leaks / path spoofing
```

---

## 4. CLI Cheat Sheet

| Task | Platform | Command |
|---|---|---|
| Configure RTR server (primary) | IOS-XR | `router bgp ASN` / `rpki server x.x.x.x` / `transport tcp port 8282` |
| Configure RTR server refresh interval | IOS-XR | `refresh-time N` (under `rpki server`) |
| Configure RTR server | IOS-XE | `router bgp ASN` / `bgp rpki server tcp x.x.x.x port 8282` |
| Match Valid routes in policy | IOS/IOS-XR | `route-map NAME permit N` / `match rpki valid` |
| Match Invalid routes in policy | IOS/IOS-XR | `route-map NAME permit N` / `match rpki invalid` |
| Match NotFound routes in policy | IOS/IOS-XR | `route-map NAME permit N` / `match rpki notfound` |
| Deny Invalid routes (hard-drop) | IOS/IOS-XR | `route-map NAME deny N` / `match rpki invalid` |
| Depref Invalid routes | IOS/IOS-XR | `route-map NAME permit N` / `match rpki invalid` / `set local-preference 50` |
| Apply RPKI policy to neighbor | IOS/IOS-XR | `neighbor x.x.x.x route-map RPKI-POLICY in` |
| View RPKI-validated prefix table | IOS/IOS-XR | `show bgp rpki table` |
| View per-prefix RPKI state | IOS/IOS-XR | `show bgp x.x.x.x/N` (look for Validation state) |
| Verify RTR session status | IOS/IOS-XR | `show bgp rpki server summary` |
| Deploy Routinator validator (Linux) | N/A | `routinator server --rtr 0.0.0.0:8282 --http 0.0.0.0:8323` |
