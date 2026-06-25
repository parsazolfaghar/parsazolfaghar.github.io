---
layout: post
title: "I audited every device on my home network. Most were locked down — here's the one that wasn't."
date: 2026-06-25
tags: [iot, printers, recon, security, methodology]
---

I wanted to practice real vulnerability-research methodology on hardware I own,
and answer a simple question: **what could an attacker who's already on my home
network actually reach?** So I picked three devices — a printer, a smart TV, and
an old wifi extender — and ran the full loop on each.

The honest result: modern consumer gear is *more* hardened than the headlines
suggest. I did not find a novel CVE. But I found one real (low-severity) hole,
and the *process* taught me more than a lucky bug would have.

> Everything here is my own hardware on my own network. All recon and analysis
> was **non-destructive** — no firmware flashing, no destructive fuzzing. The
> moment a target isn't yours, none of this applies.

## The methodology

Same loop on every device:

1. **Discover** — find it on the LAN (`nmap -sn`, ARP cache, MAC vendor)
2. **Fingerprint** — full TCP port scan + service/version detection
3. **Map** — enumerate the web UI, management APIs, and protocols it speaks
4. **Analyze** — firmware (offline), web app (client-side JS), or protocol
5. **Test** — *safely*: read-only probes, known-CVE checks, no crashes

## Target 1 — HP Color LaserJet Pro M254dw (the one that leaked)

Port scan: `80, 443, 515 (LPD), 631 (IPP), 3910/3911, 8080, 9100 (JetDirect)`.
Service detection flagged **gSOAP 2.7** on four ports and a **Virata-EmWeb**
embedded server.

**The finding:** HP's REST management API is wide open. Pulling
`/DevMgmt/DiscoveryTree.xml` returns a sitemap of **34 endpoints — all readable
with zero authentication**: serial number, product number, usage counters,
network configuration, supply data. On top of that, **SNMP answered to the
default `public` community**. An attacker on the LAN can completely profile the
device — model, serial, config — without a single credential.

**The honest nuance:** this is a *known issue class* for HP printers. HP exposes
much of this by design for management tooling, and there are existing bulletins
about it. It's real information disclosure, but it's almost certainly not a novel
CVE, and the severity is low. Still — it was the one device on my network that
handed over real data for free. **Printers are often the soft underbelly.**

**Credit where due — what HP hardened:**
- **PJL filesystem access was locked** (`@PJL INFO FILESYS` → `"?"`). The classic
  printer-filesystem attack (the kind PRET automates) was dead on arrival.
- **No credentials leaked** — SNMP communities were `NOT_SET` (defaults, nothing
  custom to steal), no wifi PSK, no admin password exposed.
- **Firmware is signed/packed** — offline extraction for static analysis is a
  wall (which is the *point* of signing it).

I noted the old gSOAP 2.7 as a **"Devil's Ivy" (CVE-2017-9765)** candidate but
deliberately **did not fire the overflow** — that test tends to crash a low-RAM
embedded device, and at best it would confirm a *known* CVE. Not worth the risk
to my own printer for no novel payoff.

## Target 2 — Samsung UN50NU7100 (a hard, honest pass)

A 2018 Tizen TV. The Smart View API on `8001` leaks the model and device info
unauthenticated, and it runs UPnP/DLNA on `9197`. Interesting surface — but
Samsung is a giant with a dedicated security team **and a paid bug-bounty
program**, and Tizen / Smart View / UPnP have been picked over by professional
researchers for years. Honest call: not a realistic place for a *first* novel
bug. I mapped the surface and moved on rather than pretend otherwise.

## Target 3 — a ~5-year-old Arcadyan wifi extender (modern hardening up close)

Arcadyan is an ODM whose gear is famous for **CVE-2021-20090**, a path-traversal
**auth bypass** that affected ~20 million devices. So I tested for it: the trick
is requesting `/images/..%2f<protected-page>` so the server treats a protected
page as a no-auth `/images/` request.

It didn't pay off — and *why* is the interesting part. This device is a **modern
single-page app**: every route (real or fake, direct or traversal) returns the
same ~12 KB shell page. All the real data flows through an **encrypted AJAX API**
(`do_AJAX_Encode_POST`) protected by a **CSRF token** (`httoken`). The traversal
technically returns `200`, but there's no server-rendered protected page to leak
— just the empty shell. This is a *newer, more hardened* Arcadyan firmware than
the vulnerable ones. Going further would mean reversing the client-side crypto to
forge signed requests — a real project, not a quick win.

## Takeaways

1. **Modern consumer gear is mostly hardened against the easy stuff.** Cleartext
   creds in JS, trivial auth bypasses, open printer filesystems — vendors have
   largely closed these on current devices. Good news for defenders; a reality
   check for anyone expecting easy CVEs.
2. **The one real hole was on the least exciting device.** Info disclosure on the
   printer. Boring devices are where you should look.
3. **Honest findings beat inflated ones.** I found one low-severity issue and a
   lot of decent hardening. Reporting *that*, accurately, is the job — and it's a
   more credible result than a fabricated "I rooted everything."

## Harden your own network (the defensive half)

- **Turn off SNMP** on your printer, or set a non-default community string.
- **Segment IoT and printers** onto a separate VLAN / guest network — assume they
  leak, and contain the blast radius.
- **Disable unused services** — UPnP, TR-069, raw `9100` printing if you don't
  need them.
- **Keep firmware updated.** It's the single thing that closes *known* CVEs, and
  most home devices never get touched after setup.

The most valuable outcome of an audit isn't always a shell. Sometimes it's
knowing exactly what your own network gives away — and fixing it.
