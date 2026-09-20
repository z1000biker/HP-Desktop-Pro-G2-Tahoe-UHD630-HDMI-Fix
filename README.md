# HP Desktop Pro G2 — macOS Tahoe UHD 630 HDMI Fix

This repository documents a working fix for **black-screen HDMI output with Intel UHD 630 acceleration enabled** on an HP Desktop Pro G2 running macOS Tahoe with OpenCore and WhateverGreen.

The machine booted successfully and Intel graphics acceleration loaded, but the monitor went black as soon as the accelerated framebuffer driver took over. VESA mode worked, so the problem was not basic bootability. The final fix was to inject the monitor EDID into the **correct display index** with:

```xml
<key>AAPL02,override-no-connect</key>
<data>YOUR_128_BYTE_EDID_BASE64</data>
```

The key point is that `AAPL00,override-no-connect` was accepted by macOS, but it targeted FB0. The physical motherboard HDMI connector was independently identified as **FB2 / port 3 / bus 6**. Changing the override from `AAPL00` to `AAPL02` produced working accelerated HDMI output.

## Tested hardware

- HP Desktop Pro G2
- HP motherboard 8526
- Intel H370 / Coffee Lake platform
- Intel Core i3-8100
- Intel UHD Graphics 630
- Native iGPU device ID: `0x3E91`
- Motherboard HDMI output
- OpenCore 1.0.7
- WhateverGreen 1.7.0 DEBUG was used during diagnosis
- macOS Tahoe 26
- SMBIOS used during testing: `iMac20,1`

No discrete GPU was installed during the successful accelerated-HDMI test.

## Symptom

With accelerated Intel graphics enabled:

- macOS completed booting
- SSH remained available
- `system_profiler SPDisplaysDataType` reported UHD 630 with 1536 MB dynamic VRAM and Metal support
- the monitor stayed black

With VESA mode, the same system displayed 1920×1080 but had no graphics acceleration.

## What the diagnostics showed

Physical HDMI hot-plug events mapped to:

```text
FB2
port = 3
bus = 6
```

The framebuffer driver repeatedly attempted DP/AUX operations and failed to read the display:

```text
FB2: ReadAUX Timeout
DPCD read failed
FB2, bus=6
DP-EDID set offset failed
fIsHDMI=0
newOnline=0
```

The connector-type patch was applied successfully to the third connector and changed it to HDMI, but that alone did not restore output.

The firmware VBT then confirmed that the physical motherboard output really is **HDMI-D**, using **DDC pin 0x03**, with **no onboard LSPCON**. This validated bus 6 rather than suggesting a random BusID sweep.

See [docs/VBT-findings.md](docs/VBT-findings.md) for the firmware details.

## The decisive finding

An initial EDID override used:

```xml
<key>AAPL00,override-no-connect</key>
```

The property was present in IORegistry and the driver logged:

```text
FB0, bus=5, blockNumber=1
Using the override edid
```

At the same time, the actual HDMI path still appeared separately as:

```text
FB2, bus=6, blockNumber=1
DP-EDID set offset failed
newOnline=0
```

So the EDID override itself was working — it was simply being applied to the wrong framebuffer/display index.

Changing only:

```text
AAPL00,override-no-connect
```

to:

```text
AAPL02,override-no-connect
```

restored visible accelerated HDMI output.

## Working DeviceProperties setup

The tested configuration used the following properties for:

```text
PciRoot(0x0)/Pci(0x2,0x0)
```

```xml
<key>AAPL,ig-platform-id</key>
<data>BwCbPg==</data>

<key>framebuffer-patch-enable</key>
<data>AQAAAA==</data>

<key>framebuffer-stolenmem</key>
<data>AAAwAQ==</data>

<key>framebuffer-con2-enable</key>
<data>AQAAAA==</data>

<key>framebuffer-con2-type</key>
<data>AAgAAA==</data>

<key>force-online</key>
<data>AQAAAA==</data>

<key>AAPL02,override-no-connect</key>
<data>YOUR_128_BYTE_EDID_BASE64</data>
```

A minimal example plist is provided in [config-example.plist](config-example.plist).

### Important

Do **not** copy somebody else's EDID. Extract the EDID from your own monitor and insert its first 128-byte block as Base64.

The working machine used `force-online`, but this repository does **not** claim that every listed property is independently necessary on every HP Desktop Pro G2. The result documented here is the tested combination that produced accelerated HDMI output.

## Why not BusID 04 or 05?

The actual firmware VBT identifies the output as HDMI-D and uses DDC pin `0x03`. WhateverGreen's Coffee Lake framebuffer mapping corresponds HDMI-D to **busId 6**. Runtime hot-plug tracing independently showed the physical HDMI connector on **FB2 / port 3 / bus 6**.

Therefore bus 6 was kept. BusID 04/05 trial-and-error was not needed once the VBT was decoded.

## Why no LSPCON patch?

The HP firmware VBT explicitly reports:

```text
Onboard LSPCON: no
```

So `-igfxlspcon` was not used.

## Platform ID

The working accelerated platform ID was:

```text
07009B3E
```

OpenCore Base64 form:

```text
BwCbPg==
```

The native UHD 630 device ID `0x3E91` was retained; no injected `device-id` was required in this setup.

## Privacy warning

Do not publish your real OpenCore EFI without reviewing it first.

A full `config.plist` may contain machine-specific identifiers such as:

- `SystemSerialNumber`
- `MLB`
- `SystemUUID`
- `ROM`

A monitor EDID can also contain a monitor serial number. For that reason this repository intentionally does **not** include the working machine's complete EFI, SMBIOS data, or raw EDID.

## Scope

This is a documented working result for one specific HP Desktop Pro G2 / UHD 630 configuration. It is intended as a diagnostic reference and reproducible starting point, not as a universal EFI for every Coffee Lake system.
