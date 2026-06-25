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
 >Default  , Delay ,  Expense , Error 



 cisco router uses only default metric rest are unused
Style of implementation
> narrow 
> wild
> transit

## Narrow metrics
- TLV 2: IS reachability
- TLV 128: IP internal reachability
- TLV 130: IP external reachability
- Metric: 6 bits, max 63
- Use case: legacy / small networks

## Wide metrics
- TLV 22: Extended IS reachability
- TLV 135: Extended IP reachability
- Metric: 24/32 bits, large scale
- Supports sub-TLVs (TE, attributes)
- Recommended for greenfield

## Transition
- Generate narrow + wide
- Accept narrow + wide
- Still capped by narrow metric if linked                                

Here’s the CCIE-style simple version.

## Narrow metrics

In **narrow metrics**, IS-IS uses the old-style TLVs:

- **TLV 2**: IS reachability.
    
- **TLV 128**: IP internal reachability.
    
- **TLV 130**: IP external reachability.
    

The metric field is only **6 bits**, so the maximum metric per hop is **63**. If you try to set a higher metric, IS-IS caps it at 63.[[cisco](https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html)]

## What this means

- Narrow metrics are limited in size.
    
- They work fine for small/simple designs.
    
- They are not ideal for modern large-scale networks.
    

If you redistribute a prefix, it is advertised in **TLV 130** as an external route, so other routers can distinguish it from internal prefixes.[[cisco](https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html)]

## Wide metrics

Wide metrics replace the old limited TLVs with:

- **TLV 22**: Extended IS reachability.
    
- **TLV 135**: Extended IP reachability.
    

These support much larger metric values, so they are better for modern networks and future-proof designs.[[cisco](https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html)]

## Transition mode

In **transition mode**, the router generates both narrow and wide metrics and accepts both.

Important point:

- Even though wide metrics are enabled, some platforms still keep the route tied to the narrow behavior.
    
- So metric values may still appear capped at **63** in certain transition cases.
    

## Wide metrics only

When you configure **wide metrics only**:

- The router can use larger metrics properly.
    
- This is the preferred mode for greenfield designs.
    
- It is the cleanest and most scalable option.
    

## External prefix behavior

This is the interesting part.

- On some IOS-XE platforms, a redistributed prefix may not always be marked as external in the same way when wide metrics are used.
    
- On IOS-XR, the external bit is typically preserved more clearly in **TLV 135**.
    

So in interview language:

**Wide metrics improve scaling, but platform behavior for external-prefix flags may differ between IOS-XE and IOS-XR.**

## CCIE oral answer

**Narrow metrics use TLV 2, 128, and 130, with a 6-bit metric field limited to 63. Wide metrics use TLV 22 and 135, support much larger metrics, and are preferred for modern IS-IS designs. Transition mode generates both narrow and wide metrics, while wide-only is the cleanest operational choice.**[[cisco](https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html)]

## Revision summary

- Narrow metrics = old TLVs, max metric 63.
    
- Wide metrics = new TLVs, scalable.
    
- Transition = both narrow and wide.
    
- Wide-only = best for new designs.
    
- External prefix handling can differ by platform.
    

I can also convert this into a **one-minute interview answer** and a **compare-table of narrow vs wide metrics** in the same style.


