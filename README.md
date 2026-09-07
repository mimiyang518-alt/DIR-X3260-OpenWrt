# DIR-X3260-OpenWrt

OpenWrt support, reverse engineering, MT7622 Wi-Fi fixes and MT7915 OEM LED research for D-Link DIR-X3260 A1.

**Status: KNOWN-GOOD PRODUCTION FINAL — 2026-09-06**

## Download firmware

**Latest hardware-validated release: v1.0.0**

[Download DIR-X3260 OpenWrt v1.0.0](https://github.com/mimiyang518-alt/DIR-X3260-OpenWrt/releases/tag/v1.0.0)

Release assets include:

- `DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin`
- `DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin`
- `SHA256SUMS.txt`
- `INSTALL.md` — English installation guide
- `INSTALL-ZH-CN.md` — 中文安装说明

> **Important:** The hardware-validated status applies to the exact binaries and SHA256 values listed below. A later rebuild from the same source is not automatically hardware validated.

## Installation

Before flashing, read the installation guide included with the v1.0.0 Release:

- **English:** `INSTALL.md`
- **中文:** `INSTALL-ZH-CN.md`

Quick network reminder:

- D-Link Bootloader Recovery: configure the computer on `192.168.0.x`, then open `http://192.168.0.1`
- After OpenWrt boots: use the `192.168.1.x` network; OpenWrt is available at `http://192.168.1.1`
- After testing, the computer can normally be returned to automatic DHCP

For an existing compatible OpenWrt installation, verify the sysupgrade image first with `sysupgrade -T` before flashing.

## Hardware-validated result

- MT7622 2.4 GHz: working (`OpenWRT`)
- MT7915 5 GHz: working (`OpenWRT-5G`)
- Internet: working
- Cold boot: passed
- LuCI 5 GHz disable/enable: passed repeatedly
- OEM-style MT7915 5 GHz LED lifecycle: recovered and validated

## Final validated firmware identities

```text
SYSUPGRADE=DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
SIZE=10752278
SHA256=1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71

RECOVERY=DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin
SIZE=21234012
SHA256=2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

These exact binaries were physically validated. A later rebuild is not automatically a replacement.

## Documentation

- [MT7622 2.4 GHz Root Cause and Production Fix](docs/MT7622-2G-Root-Cause-and-Fix.md)
- [MT7915 5 GHz OEM LED Fix](docs/MT7915-5G-LED-OEM-Fix.md)
- [Flash and Recovery Guide](docs/Flash-and-Recovery-Guide.md)
- [Reverse Engineering Timeline](docs/Reverse-Engineering-Timeline.md)
- [Build, Restore and Reproduce](docs/Build-Restore-and-Reproduce.md)

## Major breakthroughs

### MT7622 2.4 GHz

The failure was below normal AP configuration:

```text
mt7622-wmac 18000000.wmac: driver own failed
Message 00000010 timeout
Failed to get patch semaphore
```

OEM driver reverse engineering recovered the required WBSYS IOC, WPDMA and ownership semantics. The final solution is not a delay workaround.

### MT7915 5 GHz LED

The front-panel 5 GHz LED is not adequately modeled as a normal Linux GPIO LED. OEM reverse engineering recovered the MT7915 MCU LED command path, including logical LED index 1, source 26 and the required command lifecycle. The LED now follows the real AP/SSID lifecycle.

## Frozen source identity

```text
BRANCH=dirx3260-production-final
COMMIT=b6edaaf4e5df8596a742975741428217d24d11e3
TREE=40e472af26dbf5679a45da21f053ca0eccf3ec2a
TAG=dirx3260-production-final-20260906
```

## Important warning

`DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin` is **KNOWN BAD** and caused an immediate repeated reboot loop on physical hardware.

Compile success is not runtime validation.

## Public research scope

This repository publishes the reviewed final research documentation. Private memo material, temporary dumps, historical diagnostics and the preserved production workspace are intentionally not mirrored here.
