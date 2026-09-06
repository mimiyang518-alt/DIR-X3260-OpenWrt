# D-Link DIR-X3260 A1 — Build, Restore and Reproduce

> Production-final source and artifact preservation guide.

## Frozen source identity

```text
BRANCH=dirx3260-production-final
COMMIT=b6edaaf4e5df8596a742975741428217d24d11e3
TREE=40e472af26dbf5679a45da21f053ca0eccf3ec2a
TAG=dirx3260-production-final-20260906
TAG_OBJECT=53eb09811cb4698d362a045ad1c3855932455777
```

Target configuration:

```text
CONFIG_TARGET_mediatek=y
CONFIG_TARGET_mediatek_mt7622=y
CONFIG_TARGET_mediatek_mt7622_DEVICE_dlink_dir-x3260-a1=y
```

## Preserved release bundle

OpenWrt Git bundle:

```text
DIRX3260-A1-PRODUCTION-FINAL-20260906-OPENWRT.bundle
SHA256=b92f55ec23989e81daf3d5f4f31367f85fde26b0a9598dd42f1efb2d20ac67a8
```

Argon Git bundle:

```text
DIRX3260-A1-PRODUCTION-FINAL-20260906-ARGON.bundle
SHA256=a966127ba817d5c864e7ece67492b1b8475a066b349e1926d21b1374291c139d
```

Final archive:

```text
DIRX3260-A1-PRODUCTION-FINAL-20260906.tar.gz
SIZE=289702234
SHA256=e155535450baaa67b8808d4044bf9beecc1ce40e248357b420037fb2aece787a
```

## Validated binary identities

```text
SYSUPGRADE=DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
SIZE=10752278
SHA256=1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71

RECOVERY=DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin
SIZE=21234012
SHA256=2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

These exact binaries are the production artifacts because they were physically validated.

## Restoring from the OpenWrt bundle

A typical Git bundle restore is:

```sh
git clone /path/to/DIRX3260-A1-PRODUCTION-FINAL-20260906-OPENWRT.bundle DIRX3260-openwrt
git -C DIRX3260-openwrt checkout dirx3260-production-final
git -C DIRX3260-openwrt rev-parse HEAD
git -C DIRX3260-openwrt rev-parse HEAD^{tree}
```

Expected identities:

```text
HEAD=b6edaaf4e5df8596a742975741428217d24d11e3
TREE=40e472af26dbf5679a45da21f053ca0eccf3ec2a
```

Verify the tag as well:

```sh
git -C DIRX3260-openwrt show-ref --tags dirx3260-production-final-20260906
git -C DIRX3260-openwrt rev-parse dirx3260-production-final-20260906^{}
```

## Argon identity

The production environment preserved Argon independently:

```text
ORIGIN=https://github.com/jerrykuku/luci-theme-argon.git
HEAD=ddefe5f05ca334dba10d2d65d25ebf14e986ee88
TREE=6ec5351d4750007070691bbeca33c399888009cc
VERSION=2.4.7
RELEASE=20260824
```

Restore from the preserved Argon bundle when exact reproduction is required rather than silently substituting a newer theme revision.

## Effective mt76 source

The validated build used the effective source under:

```text
build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_mt7622/mt76-2026.03.21~018f6031
```

When auditing the MT7622 fix, inspect final effective source rather than drawing conclusions from historical patch filenames alone.

## MT7915 production patch identity

```text
package/kernel/mt76/patches/005-dirx3260-mt7915-5g-led-production-final.patch
SHA256=f6f5b1b0dfef24f6c6a78e41f8eff1609d119e062e7a922540aefe2f5260e064
```

Required compile-definition patch:

```text
package/kernel/mt76/patches/004-pass-LED-define-to-mt7915-via-ccflags-y.patch
SHA256=c443b6191560a9b630d8684a0fc955c6f9ff3d95bb3df6a32266644d6102266a
```

## Build environment preservation

The successful environment intentionally retains expensive and historically valuable state such as:

```text
build_dir
staging_dir
dl
bin
.dirx3260-debug
.dirx3260-disabled-patches
OEM disassembly
historical A/B patches
reverse-engineering reports
```

Do not casually run:

```text
git clean
make clean
make dirclean
```

against the preserved production/research workspace.

## Rebuild rule

A successful rebuild does **not** automatically replace the validated STAGE72 binaries.

Before calling a new binary production-equivalent, verify at minimum:

1. expected source commit/tree;
2. target configuration;
3. successful build;
4. image type and basic image validation;
5. no reboot loop;
6. 2.4 GHz `OpenWRT` visible/connectable/data;
7. 5 GHz `OpenWRT-5G` visible/connectable/data;
8. Internet;
9. 5 GHz LED startup behavior;
10. LuCI disable → LED OFF;
11. enable → actual SSID broadcast → LED ON;
12. repeated radio cycle;
13. cold boot.

Only after complete hardware validation should a newly built binary be considered a candidate to supersede the frozen artifacts.

## Separation of concerns

Keep these identities separate:

```text
SOURCE IDENTITY
BINARY IDENTITY
RUNTIME VALIDATION
```

Matching one does not automatically prove the other two.

## Public repository scope

This public repository contains reviewed research documentation. The large production archive, OEM binary evidence, temporary diagnostics and private research workspace are intentionally not published here.

---

[Back to project README](../README.md)
