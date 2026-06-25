---
uid:
title:
alias:
topic:
date:
tags:
status:
priority:

---

What are the types of authentication in ISIS

>     hello authentication(interface level
>     LSP authentication level-1 as well level-2(process level)

**What is the difference between hello authentication and LSP authentication**

 In hello authentication neighbor ship will not be formed

 In LSP authentication neighbor ship will be formed but LSP will not be installed in to database received from neighbor(level-1/level-2 depends on authentication level)

**What are different authentication style**

 command with password keyword is OLD style(text based only)

 command with authentication keyword is NEW style(text as well as MD5)

New style config as below
Process level 
   authentication mode text level-1
   authentication mode md5 level-2
   authentication  key-chain level-1 level-1
   uthentication  key-chain level-2 level-2

interface level
   isis authentication mode text level-1
   isis authentication mode md5 level-2
   isis authentication  key-chain level-1 level-1
  isis  authentication  key-chain level-2 level-2

old style method 
area-password TEST" for level-1
isis password TEST <level-1/level-2>
domain-password TEST" for level-2

++++++++++++++++++++++++++++++++++++++++++
## 1. Types of ISIS authentication (where it applies)

## a) Hello authentication (interface level)

- Applied under the **interface**.
    
- Protects **ISIS Hello (IIH) packets only**.
    
- If passwords/keys do not match:
    
    - Neighborship is **not formed at all**.
        
    - So, no adjacency, no LSP exchange.
        

Use‑case (why):

- Stops **unauthorized routers** from even joining the ISIS domain on that link.
    
- Good at security boundaries (access → aggregation) and where you want hard neighbor control.
    

---

## b) LSP authentication (process / level‑1 & level‑2)

- Applied at the **ISIS process level** for level‑1 and/or level‑2.
    
- Authenticates **LSPs (and optionally SNPs)**, not hellos.
    
- If passwords/keys do not match:
    
    - Neighborship **still comes up** (hellos are fine).
        
    - But **LSPs from that neighbor are rejected** and not installed in the LSDB (for that level).
        

Use‑case (why):

- Protects the **integrity of the link‑state database**; a neighbor can’t inject fake topology or prefixes.
    
- Very important in large SP cores and when doing SR/SRv6, because all SIDs/locators are carried in LSP TLVs.
    

---

## 2. Difference: hello vs LSP authentication

You already captured it well; here’s a crisp way to say it:

- **Hello authentication**
    
    - Controls **adjacency formation**.
        
    - Wrong key → **no neighbor relationship** at all.
        
    - Security focus: “Who is allowed to be my ISIS neighbor?”
        
- **LSP authentication**
    
    - Controls **database acceptance**.
        
    - Neighbor comes up, but wrong key → **LSPs discarded**, routes from that neighbor never appear.
        
    - Security focus: “Whose topology information do I trust?”
        

CCDE‑style explanation:

- Use hello authentication to **control membership** in the domain.
    
- Use LSP authentication to **protect control‑plane data** once membership is allowed.
    

---

## 3. Authentication styles (old vs new)

## a) Old style (text only)

Commands (Cisco‑style terminology, as you wrote):

- `area-password TEST` → L1 area authentication (LSPs in that area).
    
- `domain-password TEST` → L2/domain authentication (LSPs in the domain).
    
- `isis password TEST <level-1 | level-2>` at interface → hello authentication for that level.
    

Characteristics:

- Only **clear text** (simple) authentication.
    
- Different commands for interface (hellos) vs area/domain (LSPs).
    
- Still seen in older configs; useful to recognize for migrations.
    

## b) New style (text or MD5, key‑chains)

Process level (instance‑wide / per level):

- `authentication mode text level-1`
    
- `authentication mode md5 level-2`
    
- `authentication key-chain level-1 <name>`
    
- `authentication key-chain level-2 <name>`
    

Interface level:

- `isis authentication mode text level-1`
    
- `isis authentication mode md5 level-2`
    
- `isis authentication key-chain level-1 <name>`
    
- `isis authentication key-chain level-2 <name>`
    

Characteristics:

- Supports both **clear text** and **HMAC‑MD5** (and on some platforms, newer hashes).
    
- Uses **key‑chains**, giving:
    
    - Key rotation
        
    - Different keys per level
        
    - Better integration with security policies
        
- Conceptually splits into:
    
    - **Interface authentication** (hellos) via `isis authentication ...`
        
    - **Instance authentication** (LSPs/SNPs) via `authentication ...` in router mode.
        

Why we prefer new style (CCDE view):

- Aligns with modern security requirements (MD5/HMAC, key‑chains).
    
- Easier to standardize: one model for both IPv4/IPv6, multi‑vendor.
    
- Better for gradual migration (you can validate or send‑only on some platforms).
    

---

## 4. How to explain this in one “story”

If you need a short explanation to give in an interview:

- “ISIS supports authentication on **hellos** and on **LSPs**. Hello authentication is configured at the interface and prevents unauthorized routers from even forming an adjacency. LSP authentication is configured per level at the process; neighbors still form, but if the LSP key doesn’t match, their LSPs never enter our LSDB. Old style commands use `area-password`, `domain-password`, and `isis password` with clear text only. New style uses `authentication mode` / `isis authentication mode` plus key‑chains, supporting both text and MD5, and allows clean separation of interface vs instance authentication.”