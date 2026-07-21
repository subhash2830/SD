# 📘 DNS — Domain Name System (Design Notes + Memory Map)

## 🔹 Why DNS Exists (The Problem)
**Problem:** Computers route traffic using IP addresses, but humans can't reliably remember or type
numbers like `142.250.72.46` for every site they visit. Something needs to translate friendly names into
the addresses machines actually need.

**Design Analogy:** DNS is the internet's phone book — you look up a name, and it hands you the number to
actually dial.

**Design Principle:** DNS is a **hierarchical, distributed** database, deliberately designed with no
single central authority holding all records — this is what lets it scale to the entire internet without
any single point of failure or bottleneck.

---

## 🔹 How DNS Resolution Works
When you type `www.example.com`, here's the chain:

1. **Resolver (client-side)** — usually built into your OS/browser — checks its local cache first.
2. **Recursive Resolver** — if not cached, your query goes to a recursive resolver (your ISP's, or a
   public one like `8.8.8.8` or `1.1.1.1`). This server does all the legwork on your behalf.
3. **Root Server** — the resolver asks one of the 13 logical root server clusters: "who handles `.com`?"
   Root servers don't know the answer, just where to look next.
4. **TLD Server** — the `.com` TLD server is asked: "who is authoritative for `example.com`?"
5. **Authoritative Server** — the server that actually holds `example.com`'s real DNS records answers with
   the IP address (an A record).
6. **Response** — the recursive resolver returns the answer to your browser, and caches it for next time.

👉 **Design Insight:** This is a **recursive query** from the client's point of view (it asks once, gets a
final answer) but an **iterative process** on the resolver's side (it walks root → TLD → authoritative,
one hop at a time). Interviewers love this distinction.

---

## 🔹 DNS Record Types
| Record | Purpose |
|---|---|
| **A** | Domain → IPv4 address |
| **AAAA** | Domain → IPv6 address |
| **CNAME** | Alias — one domain name points to another domain name |
| **MX** | Mail server(s) for the domain, with priority |
| **NS** | Which servers are authoritative for this domain |
| **PTR** | Reverse lookup — IP address → domain name |
| **SOA** | Zone's administrative info — primary NS, admin contact, refresh/retry/expire timers |

---

## 🔹 Caching and TTL
- Every DNS record carries a **TTL (Time To Live)** — how long it may be cached before it must be
  re-fetched.
- Caching happens at multiple layers at once — the browser, the OS, and the recursive resolver all cache
  independently.
- **Design trade-off:** short TTL = faster failover/change propagation but more DNS query load; long TTL =
  less load but slower to propagate a change (e.g., during a migration or an incident, a long TTL can mean
  users keep hitting the old server for hours).

---

## 🔹 DNS Security
- **Cache Poisoning** — an attacker injects a forged record into a resolver's cache, silently redirecting
  users to a malicious server for that domain until the TTL expires.
- **DNS Spoofing / Man-in-the-Middle** — intercepting and forging responses in transit.
- **DDoS on DNS infrastructure** — flood an authoritative server so nobody can resolve that domain at all
  (a very effective way to take a service offline without touching it directly).
- **DNSSEC** — adds cryptographic signatures to DNS records, so a resolver can verify a response actually
  came from the real authoritative source and wasn't tampered with. Doesn't encrypt anything — it's about
  **authenticity**, not privacy.
- **Anycast + rate limiting** — the standard operational defenses against DDoS: the same IP is announced
  from many locations (see below), and abusive query rates get throttled.

---

## 🔹 Anycast DNS
The same IP address is advertised from multiple physical locations worldwide (using BGP). A query is
automatically routed to whichever instance is "closest" in the routing sense.

- **Benefit 1 — Latency**: users get answered by the nearest site.
- **Benefit 2 — Resilience**: if one site goes down or gets DDoS'd, traffic simply routes to the
  next-closest surviving site — no manual failover needed.
- This is exactly how the 13 root server "addresses" actually serve traffic from hundreds of physical
  locations globally.

---

## 🔹 DNS in Enterprise / Large-Scale Design
- **Private DNS zones** — internal-only domain names resolved by internal resolvers, never exposed to the
  public internet — standard for corporate resource naming.
- **DNS-based load balancing** — a single name resolves to multiple IPs (or uses health-checked routing
  policies), spreading client load across servers/regions without the client needing to know.
- **Split-horizon / split-brain DNS** — the same domain name returns *different* answers depending on
  whether the query came from inside or outside the corporate network — common for giving internal users a
  more direct path to internal resources.
- **Cloud DNS services** — Amazon Route 53, Azure DNS, Google Cloud DNS all add health-checked, geo-aware,
  weighted routing policies on top of plain DNS — worth namedropping in an interview to show you understand
  DNS beyond the RFC-level basics.

---

## 🔹 Troubleshooting Toolkit
```
nslookup example.com          # Basic query tool, available almost everywhere
dig example.com               # More detailed query tool (preferred by most engineers)
dig example.com +trace        # Walks the full root → TLD → authoritative chain manually
traceroute <resolved-ip>      # Once you have the IP, check if the network path itself is the problem
```
Common failure patterns:
- **No response / timeout** → resolver unreachable, firewall blocking UDP/TCP 53, or authoritative server
  down.
- **Wrong/stale answer** → cache poisoning, stale TTL, or a recently-changed record that hasn't propagated
  yet.

---

## 🔹 Key Design Takeaways
- DNS = the internet's distributed, hierarchical name-to-address directory.
- Resolution chain: Client → Recursive Resolver → Root → TLD → Authoritative → back to client.
- TTL is a real design trade-off between propagation speed and query load.
- DNSSEC = authenticity (is this response real?), not privacy (it's not encrypted).
- Anycast is what makes global DNS both fast and resilient at the same time.

---

## 🎤 Interview-Ready Answer
"DNS is a hierarchical, distributed name-to-IP directory with no single point of failure by design — a
recursive resolver walks root, then TLD, then authoritative servers on the client's behalf, and caches the
answer according to its TTL. Record types like A, CNAME, MX, and NS each serve a specific purpose in that
hierarchy. DNSSEC adds authenticity — proving a response is genuine — without adding encryption, and
Anycast is what makes global DNS both fast and resilient, by announcing the same IP from many locations and
letting BGP route each query to the nearest one."

## 🧠 Memory Map (for recall)
```
DNS → Distributed name → IP directory
   │
   ├─ Chain → Client → Recursive Resolver → Root → TLD → Authoritative → Answer
   ├─ Records → A, AAAA, CNAME, MX, NS, PTR, SOA
   ├─ Caching → TTL-driven, multi-layer (browser/OS/resolver)
   ├─ Security → Cache poisoning, spoofing, DDoS → mitigated by DNSSEC + Anycast + rate limiting
   ├─ Resilience → Anycast (same IP, many locations, BGP routes to nearest)
   └─ Enterprise → Private zones, split-horizon DNS, DNS load balancing, Route 53 / Azure DNS
```
