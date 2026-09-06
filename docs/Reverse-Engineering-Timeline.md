# D-Link DIR-X3260 A1 — Reverse Engineering Timeline

> Project closeout timeline for the OpenWrt / MT7622 / MT7915 investigation completed on 2026-09-06.

## 1. OEM firmware structure

The project began by decoding the D-Link firmware packaging and validating the OEM image structure. The SHRS container, cryptographic metadata, FIT payload, kernel, device tree and SquashFS layout were investigated sufficiently to support safe OpenWrt bring-up and recovery work.

## 2. OpenWrt boots, but the radios diverge

Early OpenWrt runtime testing showed a major split:

- MT7915 5 GHz could operate;
- integrated MT7622 2.4 GHz failed to become a usable AP.

Important errors included:

```text
mt7622-wmac 18000000.wmac: driver own failed
Message 00000010 timeout
Failed to get patch semaphore
```

This moved the investigation below LuCI/hostapd configuration and into firmware download, DMA and MCU ownership.

## 3. OEM MT7622 driver disassembly

The preserved OEM `mt7622_mt_wifi.ko` became the primary reference. Functions such as `DriverOwn`, `FwOwn`, `MakeFWOwn`, `MCUSysPrepare`, `MCUSysInit`, `MtCmdPatchSemGet` and `mt7622_trigger_intr_to_mcu` were located and disassembled.

A long series of diagnostic builds tested ownership, WPDMA state, interrupt behavior, RX pre-configuration, IOC setup and timing hypotheses.

## 4. WBSYS IOC breakthrough

OEM platform-state analysis identified the required Wi-Fi IOC configuration:

```text
infracfg base = 0x10000000
offset         = 0x320
physical       = 0x10000320
bit            = BIT(31)
```

This became part of the final MT7622 bring-up path.

## 5. WPDMA state recovered

Two final production details were established:

```text
MT_WPDMA_GLO_CFG_CLK_GATE_DIS = BIT(30)
MT_WPDMA_RX_PRE_CFG           = 0x0f7f0000
```

These changes were evaluated together with MCU/ownership behavior rather than treated as isolated magic values.

## 6. DriverOwn / FirmwareOwn semantic breakthrough

The most important MT7622 finding was that generic macro naming obscured the OEM LPCR semantics.

Final DriverOwn:

```text
trigger ON
write LPCR BIT(0)
wait BIT(0) == 1
trigger OFF
```

Final FirmwareOwn:

```text
trigger ON
write LPCR BIT(1)
wait BIT(0) == 0
trigger OFF
```

A diagnostic 500 ms pre-DriverOwn delay was later removed, proving that the production solution was not simply “wait longer”.

## 7. MT7622 2.4 GHz restored

After the IOC/WPDMA/ownership corrections, the integrated radio reached the required production behavior:

```text
OpenWRT
```

became visible, connectable and able to transfer Internet traffic, including after cold boot.

## 8. MT7915 5 GHz LED investigation

With networking working, attention moved to the front-panel 5 GHz LED. GPIO and Linux LED-trigger approaches did not reproduce the OEM lifecycle.

OEM MT7915 driver analysis revealed the MCU LED mechanism.

Recovered essentials:

```text
logical LED index = 1
source             = 26
EXT_ID             = 0x17
```

Recovered packets:

```text
MAP  02 05 01 00 00 34 00 00
OFF  02 01 01 00 00 00 00 00
ON   02 01 01 01 00 00 00 00
```

## 9. STAGE55 failure

An OEM-exact LED experiment compiled but caused an immediate reboot loop:

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

The router was recovered through the D-Link recovery path. This became a critical reminder that compile success and apparently correct reverse-engineered packets do not prove runtime safety.

## 10. Production MT7915 LED implementation

The LED implementation was consolidated into a production-safe board-specific MCU path. Debug controls and temporary instrumentation were removed.

Final behavior was tied to the real AP lifecycle:

```text
5 GHz AP active  → LED ON
5 GHz disabled   → LED OFF
AP starts again  → LED ON
```

## 11. Internet regression caught by hardware testing

During later cleanup/LED iterations, a temporary Internet regression was detected because the user explicitly tested Internet rather than assuming Wi-Fi association meant full networking was healthy. The project returned to the known-good network baseline before final consolidation.

This added Internet connectivity as a mandatory release gate.

## 12. Final hardware acceptance

The final candidate passed repeated real-device tests:

- no reboot loop;
- `OpenWRT` 2.4 GHz;
- `OpenWRT-5G` 5 GHz;
- Internet;
- correct 5 GHz LED startup behavior;
- LuCI disable → OFF;
- LuCI enable → SSID genuinely active → ON;
- client reconnect/data;
- repeated radio cycling;
- cold power cycle.

## 13. Production freeze

Final source identity:

```text
BRANCH=dirx3260-production-final
COMMIT=b6edaaf4e5df8596a742975741428217d24d11e3
TREE=40e472af26dbf5679a45da21f053ca0eccf3ec2a
TAG=dirx3260-production-final-20260906
```

Final sysupgrade SHA256:

```text
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

Final recovery SHA256:

```text
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

## 14. Project lesson

The successful method was evidence-driven iteration:

```text
runtime symptom
→ focused instrumentation
→ OEM binary comparison
→ minimal hypothesis
→ test firmware
→ physical validation
→ remove diagnostics
→ revalidate
→ freeze exact source and binaries
```

The preserved research workspace remains intentionally separate from this reviewed public documentation.

---

[Back to project README](../README.md)
