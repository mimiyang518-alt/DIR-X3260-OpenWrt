# D-Link DIR-X3260 A1 — Flash and Recovery Guide

> Applies to the hardware-validated 2026-09-06 production-final firmware.

## Validated firmware identities

### Sysupgrade

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
SIZE=10752278
SHA256=1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

### Recovery

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin
SIZE=21234012
SHA256=2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

Verify the SHA256 before flashing.

## PC network addressing

The two boot environments use different subnets:

| Router mode | PC IPv4 |
|---|---|
| D-Link recovery mode | manual `192.168.0.xxx` |
| Normal OpenWrt | `192.168.1.xxx` when manual addressing is required |
| After testing | restore automatic DHCP |

A common source of confusion is remaining on `192.168.0.xxx` after the router has left recovery and booted OpenWrt.

## OpenWrt sysupgrade

Copy the validated sysupgrade image to `/tmp`, then test it first:

```sh
sysupgrade -T /tmp/DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
```

Only proceed if the image test succeeds.

For a clean upgrade without retaining the previous configuration:

```sh
sysupgrade -n /tmp/DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
```

Do not interrupt power while flash writing is in progress.

## Recovery path

A recovery route was physically used during development after experimental firmware failures:

1. Enter the D-Link recovery environment.
2. Set the PC Ethernet interface manually to the `192.168.0.xxx` subnet.
3. Upload a known-good recovery image through the recovery interface.
4. Allow the router to complete recovery and boot.
5. When normal OpenWrt is running, move the PC to the `192.168.1.xxx` network if static addressing is required.
6. Upload the known-good production sysupgrade image.
7. Run `sysupgrade -T` before flashing.
8. Flash the production image.
9. After validation, return the PC IPv4 configuration to DHCP.

## Known-bad warning

Do **not** treat this image as production-safe:

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

It caused an immediate repeated reboot loop on physical hardware and is classified **KNOWN BAD**.

## Post-flash validation checklist

The production release was accepted only after all of the following passed:

1. No reboot loop.
2. `OpenWRT` 2.4 GHz appears and clients can connect.
3. `OpenWRT-5G` appears and clients can connect.
4. Internet works.
5. After stable startup, the 5 GHz LED turns on.
6. LuCI disable 5 GHz → SSID disappears → LED completely off.
7. LuCI enable 5 GHz → wait for `OpenWRT-5G` to genuinely broadcast → LED on.
8. A client reconnects to 5 GHz and transfers data normally.
9. Repeat disable → OFF → enable → ON.
10. Perform a cold power cycle and repeat the essential Wi-Fi/Internet/LED checks.

## Release rule

A newly compiled image is not automatically equivalent to the validated production binary. Any replacement candidate must receive the complete hardware test again.

---

[Back to project README](../README.md)
