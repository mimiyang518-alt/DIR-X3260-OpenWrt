# 从一台不能正常工作的路由器，到 Production Final
## DIR-X3260 OpenWrt 逆向工程全记录

> 设备：D-Link DIR-X3260 A1  
> SoC：MediaTek MT7622  
> 5 GHz：MediaTek MT7915  
> 项目状态：KNOWN-GOOD PRODUCTION FINAL  
> 本文定位：项目回顾，而不是刷机说明书或单一 bug 技术文档

---

## 1. 开始时，我们甚至还没有资格讨论“修 Wi-Fi”

这个项目最初面对的不是一个已经能够稳定运行 OpenWrt、只差几个设备树节点的普通移植。

摆在面前的是一台 D-Link DIR-X3260 A1，以及厂商自己的固件格式、启动链、分区、无线驱动和恢复机制。

真正的问题是一连串的：

```text
OEM firmware 到底是什么格式？
        ↓
能不能安全拆开？
        ↓
能不能重新封装？
        ↓
OpenWrt image 应该怎样进入设备？
        ↓
刷坏以后怎么回来？
        ↓
为什么 5 GHz 能工作而 2.4 GHz 不工作？
        ↓
为什么 Wi-Fi 工作了，5 GHz LED 还是不对？
        ↓
怎样证明最终 firmware 真的可以长期保留？
```

因此，这个项目最后成为了一次完整的 firmware reverse-engineering、driver reverse-engineering、hardware validation 和 release engineering 工作。

---

## 2. 第一层：先理解 D-Link OEM firmware

DIR-X3260 OEM firmware 外层使用 D-Link SHRS container。

分析最终恢复出：

```text
SHRS header
    ↓
encrypted payload
    ↓
AES-128-CBC
    ↓
FIT image
    ↓
kernel + DTB + squashfs rootfs
```

header 中还包含：

```text
salt
SHA-512 digests
4096-bit RSA public key
before / post signatures
```

OEM rootfs 中的：

```text
/bin/imgdecrypt
/etc/public.pem
/etc/enk.txt
```

为理解厂商校验路径提供了重要证据。

项目最终完成了一个非常关键的 milestone：

> **OEM image 可以被解析，并且 round-trip 后的 digest/signature 关系可以验证。**

这意味着后面的 OpenWrt 工作不再建立在“猜这个二进制文件大概是什么”的基础上。

---

## 3. Recovery 比第一个成功启动更重要

路由器 firmware 开发最危险的错误之一，是只设计“怎么刷进去”，不设计“刷坏以后怎么回来”。

DIR-X3260 的工作很早就把 recovery path 当成第一等公民。

实际形成了两套清楚的网络环境：

### D-Link recovery mode

```text
PC manual IPv4:
192.168.0.xxx
```

### 正常 OpenWrt / sysupgrade

```text
PC manual IPv4:
192.168.1.xxx
```

测试结束后：

```text
restore PC IPv4 to DHCP / automatic
```

这看似只是一个网络配置细节，后来却多次避免了把“PC 还停留在错误网段”误判成“路由器又死了”。

更重要的是，recovery path 后来真的救了项目。

---

## 4. OpenWrt 第一次运行，并不代表设备已经“支持”

早期 OpenWrt 成功启动以后，Internet、LuCI、sysupgrade 等基础能力逐步建立。

但无线状态非常不对称：

```text
MT7915 5 GHz
    → 可以工作

MT7622 integrated 2.4 GHz WMAC
    → 不能正常工作
```

2.4 GHz 的典型日志：

```text
mt7622-wmac 18000000.wmac: driver own failed
```

更早还出现：

```text
Message 00000010 timeout
Failed to get patch semaphore
```

此时最容易做的事，是继续调整：

```text
firmware
timeout
delay
hostapd
wireless UCI
```

而这个项目最重要的技术转折，就是逐渐证明：问题不在这些上层配置。

---

## 5. MT7622：从“等久一点”到重新理解 ownership

调查 MT7622 时做过大量 timing 和 diagnostic experiment：

```text
FWDL ring
MCU queue / PQID
patch semaphore
register snapshots
DriverOwn timing
500 ms pre-DriverOwn delay
1 / 5 / 20 ms trigger timing
```

500 ms delay 一度改变症状。

但最后它被从 production 中删除，而 2.4 GHz 仍然稳定工作。

这证明：

> **delay 是诊断工具，不是最终根因。**

真正的突破来自反汇编 D-Link OEM `mt7622_mt_wifi.ko`，追踪：

```text
DriverOwn
FwOwn
MakeFWOwn
MCUSysPrepare
MtCmdPatchSemGet
mt7622_trigger_intr_to_mcu
mt_load_patch
mt_load_fw
```

问题从：

```text
“为什么 DriverOwn timeout？”
```

变成：

```text
“OEM 成功执行 ownership transaction 之前，
platform 和 DMA 到底处于什么状态？”
```

---

## 6. MT7622 最终恢复的是一条初始化链

production 中最终保留的关键语义包括：

### WBSYS / IOC

```text
infracfg base    = 0x10000000
offset           = 0x320
physical address = 0x10000320
bit              = BIT(31)
```

### WPDMA

```text
MT_WPDMA_GLO_CFG_CLK_GATE_DIS = BIT(30)
MT_WPDMA_RX_PRE_CFG           = 0x0f7f0000
```

### DriverOwn

```text
trigger ON
LPCR write BIT(0)
wait BIT(0) == 1
trigger OFF
```

### FirmwareOwn

```text
trigger ON
LPCR write BIT(1)
wait BIT(0) == 0
trigger OFF
```

因此最终修复不能被简化成：

```text
add 500 ms delay
```

也不能被简化成：

```text
set BIT30
```

更准确的描述是：

```text
WBSYS / IOC
        ↓
WPDMA state
        ↓
AP → CONN trigger
        ↓
LPCR ownership semantics
        ↓
MCU / firmware
        ↓
2.4 GHz operational
```

这是整个项目最重要的技术突破。

---

## 7. 一个同样重要的结论：不要把板级证据扩大成 SoC 定律

production 解决以后，我们又重新审查了这些修改是否适合 upstream。

这个过程反而迫使我们收紧很多早期表述。

例如 BIT30：

```text
DIR-X3260 runtime requirement
    → strong evidence

all MT7622 boards require it
    → NOT proven
```

后续 OEM provenance audit 也没有证明：

```text
OEM WfHifHwInit definitely programs BIT30
```

所以最终 upstream architecture 不再使用简单的：

```c
if (is_mt7622(...))
```

而是设计成：

```text
detect DIR-X3260 once
        ↓
cache board quirk
        ↓
DMA consumes cached quirk
```

这件事非常重要，因为 reverse engineering 最危险的不是“什么都不知道”，而是：

> **已经知道一部分，然后把这一部分解释得过大。**

---

## 8. 2.4 GHz 修好以后，5 GHz LED 又暴露出另一类问题

MT7915 5 GHz radio 本身可以工作：

```text
OpenWRT-5G visible
association works
data works
Internet works
```

但 5 GHz LED 的 OEM 行为没有恢复。

早期思路自然是：

```text
Linux LED
    ↓
GPIO
    ↓
front-panel LED
```

调查过：

```text
gpio-leds
phy1radio
GPIO 86
legacy GPIO ioctl
GPIO v2
LED triggers
```

这些实验帮助找到硬件表象，却无法重现 OEM 生命周期。

最终 OEM MT7915 驱动告诉我们，真正的 owner 是：

```text
MT7915 MCU LED engine
```

---

## 9. MT7915：从 GPIO 走到 OEM MCU protocol

逆向恢复出：

```text
logical LED index = 1
source / GPIO     = 26
MCU EXT_ID        = 0x17
```

OEM mapping：

```text
AndesLedGpioMap(adapter, 1, 26, 0)
```

最终 exact packet family：

### MAP

```text
02 05 01 00 00 34 00 00
```

### OFF / CONTROL=0

```text
02 01 01 00 00 00 00 00
```

### ON / CONTROL=1

```text
02 01 01 01 00 00 00 00
```

production backend 使用：

```c
MCU_CMD(EXT_CID) |
FIELD_PREP(__MCU_CMD_FIELD_EXT_ID, 0x17)
```

brightness dispatch：

```c
mt7915_mcu_dirx3260_led_oem(dev, brightness ? 3 : 1);
```

这时我们终于知道“怎么控制灯”。

但项目还没有结束。

---

## 10. 会亮，不等于行为正确

真正的产品要求不是：

```text
send ON
    → LED lights
```

而是：

```text
5 GHz AP genuinely active
    → LED ON

LuCI disable 5G
    → LED OFF

LuCI enable 5G
    → wait until AP genuinely returns
    → LED ON
```

最终关键修复是在 MT7915 MCU 初始化后安装 source map，并在成功完成 AP/BSS/STA startup 后 replay：

```c
mt7915_mcu_dirx3260_led_oem(dev, 3);
```

也就是：

```text
CONTROL=1 / ON
```

最终 LED 代表的是：

```text
OpenWRT-5G really operational
```

而不是：

```text
driver happened to initialize
```

---

## 11. STAGE55：一次必须保留的失败

整个项目中最有价值的失败之一，是：

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

它：

```text
built successfully
image was structurally valid
hash matched
```

但刷入真实设备后：

```text
immediate repeated reboot loop
```

因此正式分类：

```text
KNOWN BAD
```

恢复只能通过 D-Link recovery，再回到 known-good sysupgrade。

这次失败永久改变了项目后面的 release discipline：

```text
COMPILE SUCCESS != RUNTIME SAFETY
HASH IDENTITY   != RUNTIME SAFETY
```

从此以后，“build 成功”不再被当成阶段完成。

---

## 12. Internet regression：为什么 known-good baseline 必须存在

项目后期还发生过另一类危险问题。

连续进行 production cleanup / LED 修改后，一些新 firmware：

```text
能启动
Wi-Fi 看起来存在
```

但：

```text
Internet 不通
```

这时没有继续盲目在最新 build 上修补，而是回到此前已知正常的 recovery 和 production-clean sysupgrade。

这一动作确认：

```text
hardware is recoverable
known-good baseline still works
regression was introduced later
```

这也是为什么最终 release gate 不只检查：

```text
boot
SSID
LED
```

还必须检查：

```text
actual Internet/data path
```

---

## 13. 最终硬件 gate

最终 firmware 的验收不是一次“看起来好了”。

完整实体测试包括：

```text
1. no reboot loop

2. OpenWRT 2.4G appears
   connect PASS
   data PASS

3. OpenWRT-5G appears
   connect PASS
   data PASS

4. Internet PASS

5. stable 5G
   LED ON

6. LuCI disable 5G
   SSID disappears
   LED completely OFF

7. LuCI enable 5G
   wait for real SSID broadcast
   LED ON

8. client reconnect
   data PASS

9. repeat disable → OFF → enable → ON

10. cold power cycle
    repeat boot / Wi-Fi / Internet / LED validation
```

全部通过以后，才进入 production freeze。

---

## 14. Production Final 不只是给 firmware 改个名字

最终冻结的 production source identity：

```text
branch:
dirx3260-production-final

commit:
b6edaaf4e5df8596a742975741428217d24d11e3

tree:
40e472af26dbf5679a45da21f053ca0eccf3ec2a

tag:
dirx3260-production-final-20260906
```

最终 sysupgrade：

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin

size:
10752278

SHA256:
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

最终 recovery：

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin

size:
21234012

SHA256:
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

production archive、Git bundle、manifest 和 backup 又分别做了 hash verification。

所以“Production Final”的含义是：

```text
source identity
        +
binary identity
        +
physical validation
        +
archive identity
        +
backup verification
```

而不是文件名里出现 `FINAL`。

---

## 15. 为什么后来停止 cleanup

项目完成以后，理论上还能继续：

```text
删 debug 文件
删历史 patch
删 build tree
git clean
make clean
整理目录
```

但最终明确选择：

```text
DO_NOT_GIT_CLEAN=YES
DO_NOT_MAKE_CLEAN=YES
DO_NOT_MAKE_DIRCLEAN=YES
KEEP_BUILD_WORKSPACE=YES
KEEP_RESEARCH_ARTIFACTS=YES
```

原因不是懒得整理。

而是此时：

```text
disk space is cheap
reconstructing reverse-engineering evidence is expensive
```

一个已经成功 build、已经通过实机验证、还保留完整研究证据的 workspace，本身就是项目资产。

---

## 16. 从工程项目变成可以被别人理解的研究记录

production freeze 后，工作没有立刻停止。

下一阶段是把“我们知道发生了什么”变成“别人也能理解发生了什么”。

最终整理了：

```text
MT7622 root-cause evidence
MT7622 root-cause/fix document
MT7915 OEM LED document
reverse-engineering timeline
flash/recovery guide
build/restore/reproduce guide
production README
documentation package
GitHub publication package
```

随后又建立公开 research/docs repository 和 Discussions。

这一步很重要，因为 reverse engineering 如果只剩：

```text
final patch
```

几年以后连作者本人都可能不知道：

```text
为什么这一行必须存在？
哪个值是 OEM 证据？
哪个值只是实验？
哪个 patch 曾经 reboot loop？
哪个行为经过实体测试？
```

因此，documentation 不是项目结束后的装饰，而是结果的一部分。

---

## 17. Upstream preparation：最难的是知道什么“不该提交”

完成 production 后，项目进入 upstream preparation。

这一步没有简单地把所有 downstream patch 打包提交。

相反，我们重新审查：

```text
哪些结论只证明了 DIR-X3260？
哪些能推广？
哪些 current upstream 已经解决？
哪些仍缺单变量证据？
```

最终结果：

### A1 — WPDMA BIT30

```text
STATUS:
UPSTREAM CANDIDATE

SCOPE:
DIR-X3260 ONLY
```

最终设计：

```text
no SoC-wide is_mt7622()
no new DT register-bit property
detect board once
cache internal quirk
DMA consumes cached quirk
```

并完成 current-upstream static apply verification。

但是：

```text
A1 alone != complete DIR-X3260 production fix
```

### A2 — RX_PRE_CFG

```text
STATUS:
HOLD
```

DIR-X3260 production 使用它，但不足以证明所有 MT7622 都需要它。

### IOC — infracfg 0x320 BIT31

```text
STATUS:
HOLD
```

DIR-X3260 production 有效，OEM evidence 支持这块板，但 generic MT7622 scope 没有证明。

### A3 — raw ownership sequence

```text
STATUS:
DOWNSTREAM ONLY
```

production runtime 通过，但 current upstream semantics 不同，证据不足以做 SoC-wide replacement。

### B1 — MT7915 `ccflags-y`

```text
ALREADY UPSTREAM
```

### B2 — DIR-X3260 MT7915 OEM MCU LED

```text
SEPARATE REVIEW REQUIRED
```

这可能是整个 upstream 阶段最成熟的结果：

> **不是“我们有四个修复，所以提交四个 patch”，而是“只有证据足够支撑 scope 的东西才应该进入 upstream”。**

---

## 18. 公开资料有什么用，又有什么局限

项目主要结论来自：

```text
OEM firmware
OEM binary disassembly
OpenWrt source
runtime diagnostics
A/B firmware
physical hardware validation
```

而不是依赖网上某一篇帖子。

公开资料仍然有价值，例如：

- D-Link 官方仍保留 DIR-X3260 A1 的 firmware、manual、datasheet；
- 官方 datasheet 明确列出独立 2.4 GHz / 5 GHz Wi-Fi LED；
- OpenWrt 2026 年出现过 `mediatek: add support for DIR-X3260` 的 PR/build activity。

但这些资料并不能直接回答：

```text
为什么 MT7622 DriverOwn 失败？
DIR-X3260 需要哪些 WPDMA/platform semantics？
MT7915 OEM LED exact payload 是什么？
什么时候应该 replay ON？
```

这些答案最终仍然必须从设备本身得到。

---

## 19. 这个项目真正建立起来的不是一个 patch，而是一条证据链

回头看，最重要的成果可以写成：

```text
OEM container
    ↓
safe recovery
    ↓
OpenWrt boot
    ↓
runtime failure
    ↓
instrumentation
    ↓
OEM disassembly
    ↓
hypothesis
    ↓
A/B firmware
    ↓
physical validation
    ↓
production consolidation
    ↓
freeze
    ↓
documentation
    ↓
upstream scope review
```

其中任何一环缺失，最终结论都会弱很多。

只有反汇编，没有实机：

```text
不知道 interpretation 是否正确
```

只有实机，没有 provenance：

```text
不知道为什么有效
```

只有 patch，没有 recovery：

```text
一次坏 firmware 就可能终止实验
```

只有 production，没有 freeze：

```text
无法确定最终验证的是哪一份 source/binary
```

只有 downstream success，没有 upstream scope review：

```text
容易把 board quirk 错误推广给整个 SoC
```

---

## 20. 几个最昂贵、也最值得保留的教训

### 1. Recovery path 要在需要 recovery 之前建立

STAGE55 证明了这一点。

### 2. 日志显示失败位置，不保证显示最早的根因

`driver own failed` 最终指向的是更深的 platform/DMA/ownership 链。

### 3. Delay 能改变症状，但不一定修复协议

500 ms delay 最终被删除。

### 4. GPIO 可见，不代表 GPIO 是真正 owner

MT7915 5 GHz LED 最终属于 MCU LED control path。

### 5. Build success 不是 release gate

STAGE55 是最直接的证明。

### 6. Internet 必须单独验证

“两个 SSID 都出现”仍然可能是 regression firmware。

### 7. Known-good baseline 不能随便覆盖

它是实验中的安全锚点。

### 8. Frozen binary 不因“重新编译成功”自动失效

新 binary 如果想替代 final binary，必须重新经过完整 hardware validation。

### 9. Downstream success 不等于 upstream-ready

scope 本身也需要证据。

### 10. 不确定性应该被记录，而不是被措辞掩盖

“DIR-X3260 需要 BIT30”和“所有 MT7622 需要 BIT30”是完全不同的结论。

---

## 21. 从“不能正常工作”到 Production Final

项目最开始看到的是：

```text
2.4 GHz 不工作
5 GHz LED 不对
firmware format 特殊
刷机风险很高
```

最后得到的是：

```text
OpenWRT
    → PASS

OpenWRT-5G
    → PASS

Internet
    → PASS

cold boot
    → PASS

5G LED lifecycle
    → PASS

recovery
    → established

source identity
    → frozen

binary identity
    → frozen

archive / backup
    → verified

documentation
    → published

upstream scope
    → reviewed
```

这就是从“能不能把 OpenWrt 跑起来”，走到真正的：

```text
KNOWN-GOOD PRODUCTION FINAL
```

---

## 22. 最终结语

DIR-X3260 项目最有价值的地方，不是最后生成了两个 firmware 文件。

它证明了一套在资料不足、闭源驱动存在、硬件行为不明确时仍然可以推进的方法：

```text
不要猜得比证据更远
        ↓
保留 recovery
        ↓
保留 known-good baseline
        ↓
把失败变成 evidence
        ↓
反汇编 OEM
        ↓
做最小可验证假设
        ↓
实体设备闭环
        ↓
冻结真正验证过的结果
        ↓
最后再讨论 upstream
```

MT7622 的突破来自意识到 `driver own failed` 不是一个“多等 500 ms”的问题。

MT7915 LED 的突破来自意识到“能看到 GPIO”不等于“GPIO 就是 owner”。

而整个项目最终能够进入 Production Final，则来自另一个更朴素的原则：

> **任何一个没有在真实设备上通过完整验证的结果，都不能因为代码漂亮、编译成功或 hash 正确，就被称为最终结果。**

这也是这次 DIR-X3260 OpenWrt 逆向工程最值得留下来的东西。

---

## 附：最终关键身份

```text
Production branch:
dirx3260-production-final

Production commit:
b6edaaf4e5df8596a742975741428217d24d11e3

Production tree:
40e472af26dbf5679a45da21f053ca0eccf3ec2a

Production tag:
dirx3260-production-final-20260906
```

```text
Sysupgrade SHA256:
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71

Recovery SHA256:
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

这些 identity 与实体硬件验证记录一起，定义了本项目的 Production Final。
