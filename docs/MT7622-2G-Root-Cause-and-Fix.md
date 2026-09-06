# D-Link DIR-X3260 A1 — MT7622 2.4 GHz Root Cause and Production Fix

> **Status:** FINAL — KNOWN-GOOD PRODUCTION FINAL  
> **Date:** 2026-09-06  
> **Target:** D-Link DIR-X3260 A1 / MediaTek MT7622 integrated 2.4 GHz WMAC

## Final source and release identity

| Item | Value |
|---|---|
| Production branch | `dirx3260-production-final` |
| Production commit | `b6edaaf4e5df8596a742975741428217d24d11e3` |
| Production tree | `40e472af26dbf5679a45da21f053ca0eccf3ec2a` |
| Sysupgrade | `DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin` |
| Sysupgrade size | `10,752,278` bytes |
| Sysupgrade SHA256 | `1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71` |
| Recovery | `DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin` |
| Recovery size | `21,234,012` bytes |
| Recovery SHA256 | `2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097` |

## TL;DR

The 2.4 GHz failure was **not an SSID, hostapd, LuCI or ordinary Wi-Fi configuration problem**. The failure occurred below AP configuration, in the **MT7622 Wi-Fi subsystem bring-up / WPDMA / MCU ownership path**.

The production fix reconstructs OEM-required MT7622 behavior:

1. Set the board's WBSYS IOC bit at `infracfg + 0x320` (`0x10000320`, `BIT(31)`).
2. Keep WPDMA clock ungated with `MT_WPDMA_GLO_CFG_CLK_GATE_DIS = BIT(30)`.
3. Write `MT_WPDMA_RX_PRE_CFG = 0x0f7f0000`.
4. Assert the AP-to-CONN wake/trigger **before** the LPCR ownership command.
5. Use the **actual OEM MT7622 LPCR semantics**, not the generic mt7615 macro names:
   - DriverOwn: write `BIT(0)`, wait for `BIT(0) == 1`.
   - FirmwareOwn: write `BIT(1)`, wait for `BIT(0) == 0`.
6. Remove experimental delays and diagnostic instrumentation from the final effective source.

Final hardware result: `OpenWRT` 2.4 GHz is visible, connectable, transfers data, has Internet access, and survives a cold boot.

## Failure symptoms

The key failure was:

```text
mt7622-wmac 18000000.wmac: driver own failed
```

Earlier diagnostic builds also exposed:

```text
Message 00000010 timeout
Failed to get patch semaphore
```

These errors occur below normal hostapd/AP setup.

## Evidence basis

The D-Link OEM MT7622 driver was preserved and disassembled. High-value functions included `DriverOwn`, `FwOwn`, `MakeFWOwn`, `MCUSysPrepare`, `MCUSysInit`, `MtCmdPatchSemGet`, `mt7622_trigger_intr_to_mcu`, `mt_load_patch`, `mt_load_fw` and `mt7622_init`.

The full preserved 2.4 GHz disassembly identity is:

```text
DIRX3260-MT7622-2G-FULL-DISASM.txt
SIZE   = 31943772
SHA256 = b095e5d9ffa781f92ddd2ca68f2b4aacd10666c794f13b9efb927f6c9d036840
```

The final post-patch mt76 source was inspected at:

```text
build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_mt7622/mt76-2026.03.21~018f6031
```

## WBSYS / IOC initialization

| Field | Value |
|---|---|
| infracfg base | `0x10000000` |
| Wi-Fi IOC offset | `0x320` |
| bit | `BIT(31)` |
| physical register | `0x10000320` |

Effective operation:

```c
regmap_update_bits(dev->infracfg, 0x320, BIT(31), BIT(31));
```

## WPDMA initialization

The final source defines:

```c
#define MT_WPDMA_GLO_CFG_CLK_GATE_DIS BIT(30)
```

and applies it to MT7622. The final path also writes:

```c
mt76_wr(dev, MT_WPDMA_RX_PRE_CFG, 0x0f7f0000);
```

## DriverOwn / FirmwareOwn — the central trap

### DriverOwn

```text
AP2CONN_WAKE / HIF trigger = ON
LPCR write                  = BIT(0)
wait                        = LPCR BIT(0) == 1
AP2CONN_WAKE / HIF trigger = OFF
```

### FirmwareOwn

```text
AP2CONN_WAKE / HIF trigger = ON
LPCR write                  = BIT(1)
wait                        = LPCR BIT(0) == 0
AP2CONN_WAKE / HIF trigger = OFF
```

The command bit and polled status bit are not interchangeable.

## Why the 500 ms delay was not the fix

A temporary pre-DriverOwn `500 ms` delay was used diagnostically and explicitly removed from the final production stack. Likewise, 1/5/20 ms timing probes were diagnostic only.

> The production solution is **not** “sleep 500 ms before DriverOwn.”

## Root-cause statement

> The DIR-X3260 A1 MT7622 2.4 GHz failure was a low-level MT7622 Wi-Fi subsystem initialization and ownership-handshake incompatibility between the generic OpenWrt/mt76 path and the board/OEM-required MT7622 bring-up semantics.

The required corrections span WBSYS/IOC platform configuration, WPDMA clock-gating state, WPDMA RX pre-configuration, AP-to-CONN wake/trigger ordering and MT7622-specific LPCR ownership semantics.

## Final semantics that must be preserved

```text
IOC / WBSYS:
resolve mediatek,infracfg
update offset 0x320
set BIT(31)

WPDMA:
set MT_WPDMA_GLO_CFG_CLK_GATE_DIS / BIT(30)
write MT_WPDMA_RX_PRE_CFG = 0x0f7f0000

DriverOwn:
trigger ON
write LPCR BIT(0)
wait BIT(0) == 1
trigger OFF

FirmwareOwn:
trigger ON
write LPCR BIT(1)
wait BIT(0) == 0
trigger OFF
```

Production cleanup requires no 500 ms pre-DriverOwn sleep, no timing probes and no routine diagnostic register spam.

## Final hardware validation

| Test | Result |
|---|---|
| No reboot loop | ✅ PASS |
| 2.4 GHz SSID | `OpenWRT` |
| 2.4 GHz visible/connect/data | ✅ PASS |
| 5 GHz SSID | `OpenWRT-5G` |
| 5 GHz connect | ✅ PASS |
| Internet | ✅ PASS |
| Cold boot | ✅ PASS |

The hardware-validated binary remains the release artifact. A later rebuild does **not** automatically replace it.

## Preservation rules

Preserve the final build workspace, OEM disassembly and historical A/B evidence. Do not use `git clean`, `make clean` or `make dirclean` against the archived working environment. Any semantic change requires complete hardware revalidation.

---

[Back to project README](../README.md)
