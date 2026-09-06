# DIR-X3260 的 MT7622 Wi-Fi 为什么一直起不来
## 从 `driver own failed` 到 IOC / WPDMA / Ownership 的完整根因

> 设备：D-Link DIR-X3260 A1  
> SoC：MediaTek MT7622  
> 2.4 GHz：MT7622 integrated WMAC  
> 5 GHz：MT7915  
> 状态：Production-final，已完成实体设备验证

---

## 1. 表面症状

在 DIR-X3260 A1 上启动 OpenWrt 时，5 GHz MT7915 可以工作，而 MT7622 自带的 2.4 GHz WMAC 长期无法真正进入可用状态。最醒目的错误是：

```text
mt7622-wmac 18000000.wmac: driver own failed
```

初始化过程中还出现过：

```text
Message 00000010 timeout
Failed to get patch semaphore
```

从用户空间看，这很容易被误判为 SSID、hostapd、LuCI、firmware 加载速度或者 timeout 问题。但最终证据表明，故障发生在这些层次之前：MT7622 Wi-Fi subsystem 没有按照这块板实际需要的底层 bring-up / ownership 语义完成初始化。

---

## 2. 为什么特别难定位

故障链跨越多个层次：

```text
WBSYS / infracfg
        ↓
WPDMA
        ↓
AP → CONN trigger
        ↓
LPCR ownership transaction
        ↓
MCU / firmware communication
        ↓
DriverOwn / FirmwareOwn
        ↓
mac80211 / AP
```

任何一个前置状态不正确，最后都可能只表现成一句 `driver own failed`。因此，最明显的错误日志并不等于最初的错误。

---

## 3. “继续加延时”为什么不是答案

调查早期做过 pre-DriverOwn delay、500 ms delay、1/5/20 ms timing probe、MCU timeout trace、register snapshot 和 patch semaphore trace。500 ms delay 确实改变过现象，但：

```text
timing changes symptoms
        ≠
timing is the root cause
```

最终 production 删除了 500 ms pre-DriverOwn delay、1/5/20 ms timing probes 和常规诊断 trace，而 2.4 GHz 仍然稳定工作。

因此最终结论不是“OpenWrt 等得不够久”。

---

## 4. 真正的转折：反汇编 OEM 驱动

突破来自停止继续调 timeout，转而对 D-Link OEM `mt7622_mt_wifi.ko` 做符号定位和反汇编，并追踪：

```text
DriverOwn
FwOwn
MakeFWOwn
MCUSysPrepare
MCUSysInit
MtCmdPatchSemGet
mt7622_trigger_intr_to_mcu
mt_load_patch
mt_load_fw
mt7622_init
```

核心问题变成：

1. DriverOwn 前 OEM 建立了什么平台状态？
2. WPDMA 处于什么状态？
3. AP/CONN trigger 何时打开和关闭？
4. LPCR 写哪个 bit？
5. completion condition 等哪个 bit 变成什么值？

这让 timeout 从“猜测”变成了可以逐项验证的 transaction。

---

## 5. WBSYS IOC

最终恢复出的平台条件：

```text
infracfg base    = 0x10000000
register offset  = 0x320
physical address = 0x10000320
bit              = BIT(31)
```

production 有效语义：

```text
resolve mediatek,infracfg
update offset 0x320
set BIT(31)
```

这证明问题位于正常 mac80211/AP 配置层之下。但 IOC 只是组合修复的一部分，不是单独的“魔法寄存器”。

---

## 6. WPDMA 状态

最终 production 保留：

```text
MT_WPDMA_GLO_CFG_CLK_GATE_DIS = BIT(30)
MT_WPDMA_RX_PRE_CFG = 0x0f7f0000
```

这解释了为什么症状大量集中在 MCU、patch semaphore 和 ownership 附近：如果 DMA/platform state 没建立到这块板需要的状态，仅延长 MCU timeout 不会自动重建正确 transaction。

这里必须保留证据边界：实机证明 DIR-X3260 A1 需要这些 production 语义，但没有证明所有 MT7622 板都需要完全相同的处理。后续 OEM provenance 审计也没有证明“D-Link OEM 驱动明确在同一位置设置 BIT30”。

因此：

```text
DIR-X3260 BIT30 runtime requirement = strong evidence
MT7622 SoC-wide requirement         = not proven
```

---

## 7. DriverOwn：最关键的突破

OEM 反汇编恢复出的 DriverOwn transaction：

```text
trigger ON
    ↓
write LPCR BIT(0)
    ↓
wait BIT(0) == 1
    ↓
trigger OFF
```

也就是：

```c
trigger(true);
write_lpcr(BIT(0));
wait_until(lpcr & BIT(0));
trigger(false);
```

关键在于：**trigger 本身就是 ownership transaction 的组成部分。**

它不能被简化成“写 LPCR，然后多等一会儿”。等待一个语义错误的 transaction，并不会让它自动变正确。

---

## 8. FirmwareOwn：对应的另一半

最终恢复：

```text
trigger ON
    ↓
write LPCR BIT(1)
    ↓
wait BIT(0) == 0
    ↓
trigger OFF
```

因此最终语义是：

```text
DriverOwn:
    trigger ON
    LPCR BIT(0)
    wait BIT(0) == 1
    trigger OFF

FirmwareOwn:
    trigger ON
    LPCR BIT(1)
    wait BIT(0) == 0
    trigger OFF
```

ownership 不是一个 timeout 参数，而是一套硬件 transaction protocol。

---

## 9. 为什么 MCU timeout / patch semaphore 会误导

早期错误：

```text
Failed to get patch semaphore
Message 00000010 timeout
driver own failed
```

完成 OEM 对照后，更合理的因果链是：

```text
platform / DMA state incomplete
        ↓
ownership transaction incompatible
        ↓
MCU communication cannot progress reliably
        ↓
patch / firmware initialization timeouts
        ↓
driver own failed
```

日志是真实的，但它显示的是失败位置，不一定是最早的根因位置。

---

## 10. 最终不是 BIT30 单点修复

最终 production-level 结论是：

> DIR-X3260 A1 的 MT7622 2.4 GHz 故障，是 generic OpenWrt/mt76 路径与该设备实际需要的 MT7622 platform initialization / ownership-handshake 语义之间的不兼容。

最终有效行为跨越：

```text
WBSYS / IOC
    +
WPDMA clock-gating state
    +
WPDMA RX pre-configuration
    +
AP → CONN trigger ordering
    +
MT7622 LPCR DriverOwn/FirmwareOwn semantics
```

因此最准确的描述是：

```text
combined MT7622 initialization / ownership fix
```

而不是 delay fix 或单一 BIT30 fix。

---

## 11. 从诊断 patch 到 production

调查链大致经历：

```text
FWDL ring diagnostics
        ↓
MCU / PQID diagnostics
        ↓
runtime trace
        ↓
RX pre-config test
        ↓
OEM ownership MMIO comparison
        ↓
500 ms delay experiment
        ↓
IOC reconstruction
        ↓
WPDMA reconstruction
        ↓
DriverOwn ordering / LPCR trace
        ↓
trigger timing experiments
        ↓
OEM LPCR semantics
        ↓
production log cleanup
        ↓
remove 500 ms delay
```

因此证据必须按以下顺序解释：

```text
historical experiment
        ↓
diagnostic evidence
        ↓
later override / cleanup
        ↓
final effective source
        ↓
physical hardware validation
```

patch 文件名记录调查历史；effective source 才决定最终机器执行什么。

---

## 12. 最终实体设备验证

production consolidation 后，在真实 DIR-X3260 A1 上验证：

| 项目 | 结果 |
|---|---|
| reboot loop | 无，PASS |
| 2.4 GHz SSID | `OpenWRT` |
| 2.4 GHz 可见 | PASS |
| 2.4 GHz 可连接 | PASS |
| 2.4 GHz 数据 | PASS |
| 5 GHz SSID | `OpenWRT-5G` |
| 5 GHz 可连接 | PASS |
| Internet | PASS |
| cold boot | PASS |

静态代码、patch apply 和编译不能代替实体设备验证。最终 production-final 是已经通过硬件 gate 的 binary 加冻结源码状态；以后重建的 binary 若要替代它，也必须重新完成硬件验证。

---

## 13. Upstream 阶段为什么要求“少说一点”

production 成功不等于可以把所有行为推广给整个 MT7622 SoC。

最终 upstream 分类：

```text
A1 / WPDMA BIT30
    → DIR-X3260-specific upstream candidate

A2 / RX_PRE_CFG
    → HOLD

IOC / infracfg 0x320 BIT31
    → HOLD

A3 / raw DriverOwn/FirmwareOwn semantics
    → DOWNSTREAM ONLY

B1 / mt7915 ccflags-y
    → already upstream

B2 / DIR-X3260 MT7915 OEM MCU LED
    → separate review
```

A1 最终设计也从 SoC-wide 改成：

```text
detect DIR-X3260 once during probe/init
        ↓
cache internal quirk
        ↓
DMA consumes cached quirk
        ↓
set BIT30 only for this board
```

因为我们证明的是 DIR-X3260，而不是所有 MT7622。

---

## 14. 真正解决了什么

不是：

```text
SSID 配错
hostapd 配错
LuCI 配错
等待时间太短
一个 timeout 太小
```

而是：

```text
MT7622 platform state
        +
WPDMA state
        +
AP/CONN trigger
        +
LPCR command semantics
        +
ownership completion condition
```

只有这些条件组合正确，后面的 MCU firmware、mac80211、hostapd 和 SSID 才真正有机会工作。

---

## 15. 最值得保留的工程经验

1. **最明显的错误日志不一定是最早的错误。** `driver own failed` 是突破口，不是完整根因描述。
2. **timing experiment 是诊断工具，不自动等于修复。** 500 ms delay 最终被删除。
3. **OEM binary 可以成为硬件语义证据。** 在公开资料不足时，反汇编配合 runtime A/B test 可以把猜测变成可验证 transaction。
4. **不要孤立看一个寄存器。** IOC、WPDMA、trigger 和 ownership 是同一条初始化链上的不同状态。
5. **“在我的板上有效”不是 upstream 的充分条件。** production engineering 证明已知硬件正确；upstream engineering 还必须证明 scope。
6. **最终 binary 的价值来自硬件验证。** hash、commit、patch 和 reproducible build 都不能替代 cold boot、association、data path 和 Internet 实测。

---

## 16. 最终结论

真正的突破，是把问题从：

```text
“为什么 DriverOwn 等不到？”
```

改写成：

```text
“在 OEM 成功执行 DriverOwn 之前，
硬件到底已经处于什么状态，
ownership transaction 到底是什么？”
```

答案最终落在：

```text
WBSYS IOC
    ↓
WPDMA
    ↓
AP → CONN trigger
    ↓
LPCR DriverOwn / FirmwareOwn
    ↓
MCU / firmware
    ↓
2.4 GHz radio operational
```

这条链恢复以后，500 ms delay 可以删除，诊断 trace 可以删除，2.4 GHz 仍能在冷启动后稳定出现、连接并传输数据。

> **DIR-X3260 A1 上的 MT7622 2.4 GHz 故障，是一个底层 Wi-Fi subsystem initialization 与 ownership-handshake 语义不兼容问题；最终修复来自对 OEM platform、WPDMA 和 ownership transaction 的重建，而不是增加等待时间。**

---

## 证据边界

本文不作以下超出证据的断言：

- 不宣称 OEM 已被证明在同一初始化点明确设置 WPDMA BIT30；
- 不宣称全部 MT7622 平台都需要 DIR-X3260 的所有 workaround；
- 不宣称 A1 BIT30 upstream candidate 单独就能完整修复 DIR-X3260；
- 不把历史 500 ms delay 当成 production solution；
- 不根据 patch 文件名推断最终 effective behavior。

这些限制是本项目从“能运行”走到“可以准确解释”之后的重要成果。
