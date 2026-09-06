# D-Link DIR-X3260 A1 — MT7915 5 GHz OEM LED Fix

> **Status:** FINAL — hardware validated  
> **Date:** 2026-09-06

## Summary

The DIR-X3260 A1 front-panel 5 GHz LED could not be reproduced correctly as a simple Linux GPIO LED. OEM driver reverse engineering showed that the board uses the **MT7915 MCU LED engine**.

The final production implementation reproduces the OEM MCU command path and ties LED state to the real AP lifecycle.

## Recovered OEM mapping

```text
logical LED index = 1
source/GPIO       = 26
MCU EXT_ID        = 0x17
```

OEM mapping call recovered from the binary:

```text
AndesLedGpioMap(adapter, 1, 26, 0)
```

Board DTS source:

```dts
led-sources = <26>;
```

## Recovered MCU packets

MAP:

```text
02 05 01 00 00 34 00 00
```

OFF / CONTROL=0:

```text
02 01 01 00 00 00 00 00
```

ON / CONTROL=1:

```text
02 01 01 01 00 00 00 00
```

The MCU command uses:

```c
FIELD_PREP(__MCU_CMD_FIELD_EXT_ID, 0x17)
```

The production state mapping is:

```c
brightness ? 3 : 1
```

and the implementation is board-gated to:

```text
dlink,dir-x3260-a1
```

## Runtime lifecycle

The important requirement is not merely “GPIO high/low”. The LED must follow whether the 5 GHz AP is genuinely active.

```text
boot
  ↓
MT7915 initialization
  ↓
OpenWRT-5G genuinely broadcasts
  ↓
LED ON

LuCI disable 5 GHz
  ↓
SSID disappears
  ↓
LED OFF

LuCI enable 5 GHz
  ↓
wait for AP startup / OpenWRT-5G broadcast
  ↓
LED ON
```

The production implementation includes the OEM map operation and replays the required LED state after AP startup.

## Why earlier GPIO approaches were insufficient

The investigation tested Linux LED/GPIO ownership and GPIO 86 observations, but those paths did not reproduce the OEM behavior. The decisive evidence came from the OEM MT7915 driver, Andes command construction and runtime validation.

The final solution therefore uses the MT7915 MCU/OEM command path rather than assuming the front-panel 5 GHz LED is a conventional host-controlled GPIO.

## Important failed build

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

is **KNOWN BAD**. It caused an immediate repeated reboot loop on physical hardware.

This was an important release lesson: reproducing an OEM-looking command sequence is not sufficient by itself. Driver integration, lifecycle and runtime safety must also be validated.

## Final production patch

```text
package/kernel/mt76/patches/005-dirx3260-mt7915-5g-led-production-final.patch
SHA256=f6f5b1b0dfef24f6c6a78e41f8eff1609d119e062e7a922540aefe2f5260e064
```

Required LED compile-definition patch:

```text
package/kernel/mt76/patches/004-pass-LED-define-to-mt7915-via-ccflags-y.patch
SHA256=c443b6191560a9b630d8684a0fc955c6f9ff3d95bb3df6a32266644d6102266a
```

The final production source does **not** retain manual debugfs LED test controls or diagnostic instrumentation.

## Hardware validation

The final production firmware passed:

- stable startup → 5 GHz LED ON after the AP becomes active;
- LuCI disable 5 GHz → SSID disappears and LED fully OFF;
- LuCI enable 5 GHz → `OpenWRT-5G` returns and LED turns ON;
- client reconnect and Internet/data traffic;
- repeated disable → OFF → enable → ON;
- cold power cycle with correct 5 GHz operation and LED behavior;
- no reboot loop.

## Production rule

Any future refactor must preserve the recovered MCU semantics **and** repeat the complete hardware validation. Compile success alone is not sufficient.

---

[Back to project README](../README.md)
