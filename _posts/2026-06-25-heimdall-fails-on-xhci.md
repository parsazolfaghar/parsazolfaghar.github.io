---
layout: post
title: "Why Heimdall fails to flash Samsung devices on modern AMD/Mac machines — and the one-cable fix"
date: 2026-06-25
tags: [debugging, usb, android, heimdall, firmware]
---

**TL;DR:** If `heimdall flash` dies at `Uploading … 0%` with
`ERROR: Expected file part index: 0 Received: 1`, it's almost certainly your
host's **USB controller**, not Heimdall. Modern AMD boards (and all recent
Macs) are **xHCI-only** — no legacy USB2/EHCI controller — and xHCI mishandles
Samsung's Odin bulk-OUT transfers. **Fix: plug the phone into the PC through
*any* external USB hub** (USB2 or USB3). The phone is a USB2 device, so it
routes through the hub's USB2 transaction translator, which "launders" the
timing xHCI gets wrong. The upload then sails `0% → 100%`.

> Context: authorized use only. This was on my own, expendable Samsung Galaxy
> J7 — flashing TWRP recovery from a Kali box. Don't point any of this at
> hardware that isn't yours.

---

## The setup

I wanted root on an old Galaxy J7 (SM-J700F, Android 6.0.1). The clean path is
Odin/Download mode → flash TWRP recovery → root from there. On Linux the Odin
equivalent is **Heimdall**:

```bash
heimdall flash --RECOVERY twrp-j7elte.img --no-reboot
```

Phone in Download mode, detected fine. Then:

```
Downloading device's PIT file...
PIT file download successful.
Uploading RECOVERY
0%
ERROR: Failed to receive file part response!
ERROR: Retrying...
ERROR: Expected file part index: 0 Received: 1
ERROR: RECOVERY upload failed!
```

Byte-identical, every single attempt.

## The investigation (and the mistake I didn't make)

The tempting move is to thrash on Heimdall: try flags, try a different image,
reinstall. I forced myself to gather evidence first.

**Observation 1 — it's asymmetric.** The **PIT download** (device → host)
succeeds every time. Only the **RECOVERY upload** (host → device) fails, and it
fails on the *very first packet* (`index 0`, at 0%). So enumeration, Download
mode, the protocol handshake, and one direction of bulk transfer all work.
Something is specifically wrong with **host → device bulk-OUT**.

**Observation 2 — it's not Heimdall.** Kali ships Heimdall 2.1.0. I grabbed
1.4.2 as well (extracted from an old `.deb`, no root needed) and ran the exact
same flash. **Identical failure, same packet, same error text.** When two
independent versions of a tool fail byte-for-byte at the same spot, the tool
isn't the variable. Something underneath it is.

**Observation 3 — the layer underneath is USB.** `lsusb -t`:

```
/:  Bus 001 ... Driver=xhci_hcd/10p, 480M
    |__ Port 004: ... Samsung ... (Download mode), 480M
```

The phone negotiates at 480 Mbps (USB2 high-speed), but look at the driver:
`xhci_hcd`. And `lspci`:

```
USB controller: AMD ... USB 3.1 xHCI Compliant Host Controller
USB controller: AMD ... Matisse USB 3.0 Host Controller
```

**Every controller on the board is xHCI.** Modern AMD chipsets dropped the old
EHCI (USB2-only) controllers entirely — so does every recent Mac. A USB2 device
like this phone is handled by an xHCI controller running it in compatibility
mode.

## Root cause

Heimdall's Odin-protocol bulk-OUT path was written in the EHCI era. xHCI
schedules and acknowledges bulk transfers differently, and for this particular
protocol the device's acknowledgement comes back one packet "ahead" of what
Heimdall expects — hence `Expected index 0, Received 1`, right at the first
packet. The download path (bulk-IN) isn't affected, which is exactly why the
PIT downloads fine while the upload desyncs.

This isn't a Heimdall bug you can patch around with flags, and it isn't fixed by
changing versions (proven above). It's the host controller.

## The fix: put a hub in the middle

You can't add an EHCI controller to a board that doesn't have one. But you can
change what the xHCI controller *talks to*: insert **any external USB hub**
between the phone and the PC.

Here's the key bit of USB that makes this work. A USB2 device behind a hub
doesn't talk to the host controller directly — it talks through the hub's
**Transaction Translator (TT)**. The host does high-speed transactions to the
hub; the hub's TT does the low/full/high-speed split transactions to the
device, on its own timing. That TT step re-clocks the bulk-OUT traffic and
sidesteps the xHCI scheduling quirk. (This works with a "USB3" hub too — a USB2
device still goes through the hub's internal USB2 hub silicon, which has a TT.)

With a cheap hub inline:

```
Uploading RECOVERY
0%5%10%...95%100%
RECOVERY upload successful
```

First try. The fix is literally "add one more connector to the chain."

## Two takeaways

1. **When two versions of a tool fail identically, stop debugging the tool.**
   The byte-for-byte match across Heimdall 2.1.0 and 1.4.2 was the single most
   useful data point — it ruled out the entire application layer in one move and
   pointed straight down the stack.
2. **Know the layer below the one you're working in.** "USB just works" until it
   doesn't; understanding xHCI vs EHCI and what a hub's transaction translator
   actually does turned an unfixable-looking flash failure into a one-cable fix.

If you're on a modern AMD desktop or any recent Mac and Heimdall (or Odin-likes)
choke on the upload, don't reinstall anything — **reach for a USB hub first.**
