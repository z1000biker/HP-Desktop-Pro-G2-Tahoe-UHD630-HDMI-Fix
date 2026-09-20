# VBT findings

This document records the firmware-routing evidence used to diagnose the black-screen HDMI problem on the tested HP Desktop Pro G2.

## Machine

- HP Desktop Pro G2
- HP system board: 8526
- Intel H370 / Coffee Lake
- Intel UHD Graphics 630
- PCI device ID: `8086:3E91`
- Physical display connection: motherboard HDMI

## VBT extraction

The firmware Video BIOS Table was extracted under a Linux live environment through i915 debugfs:

```bash
sudo cat /sys/kernel/debug/dri/0000:00:02.0/i915_vbt > ~/vbt.bin

sudo intel_vbt_decode \
  --file=/sys/kernel/debug/dri/0000:00:02.0/i915_vbt \
  --devid=0x3E91 | tee ~/vbt.txt
```

The decoded VBT identified itself as:

```text
$VBT COFFEELAKE
BDB version 209
```

## Relevant child-device entry

The external TMDS output was reported as:

```text
EFP3
DVI-D / TMDS
DVO Port HDMI-D (0x03)
DDC pin 0x03
Aux channel: none
Onboard LSPCON: no
HDMI level shifter: 0x08
IBoost level HDMI: 0x02
```

The important points are:

1. The physical output is **HDMI-D**.
2. Its DDC routing is **pin 0x03**.
3. It has **no AUX channel** in firmware.
4. It has **no onboard LSPCON**.

## Relation to WhateverGreen framebuffer routing

For the Coffee Lake framebuffer table used during testing, the native connector records for platform `0x3E9B0007` were:

```text
connector 0: index 1, busId 5, pipe 9, DP
connector 1: index 2, busId 4, pipe 10, DP
connector 2: index 3, busId 6, pipe 8, DP
```

The patch:

```text
framebuffer-con2-type = 00000800
```

changed the third connector to HDMI while retaining busId 6.

Runtime unplug/replug tracing independently mapped the motherboard HDMI socket to:

```text
FB2
port 3
bus 6
```

This matched the firmware's HDMI-D routing, so busId 6 was retained.

## Why the first EDID injection failed

The first override was:

```text
AAPL00,override-no-connect
```

IORegistry confirmed that the property was injected. The Intel framebuffer log then showed the override being consumed on FB0:

```text
FB0, bus=5, blockNumber=1
Using the override edid
```

Meanwhile the physical HDMI path remained:

```text
FB2, bus=6, blockNumber=1
DP-EDID set offset failed
fIsHDMI=0
newOnline=0
```

This demonstrated that the EDID data itself was accepted but the display index was wrong.

The only change in the final test was:

```text
AAPL00,override-no-connect
        ↓
AAPL02,override-no-connect
```

After that change, accelerated HDMI output became visible.

## Notes

The VBT evidence is useful because it prevents blind trial-and-error with unrelated BusIDs or LSPCON options.

The final result does not imply that every HP Desktop Pro G2 has identical firmware routing. Decode the VBT on the target machine when diagnosing a different board or firmware revision.
