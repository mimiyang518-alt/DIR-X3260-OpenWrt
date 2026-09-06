# 怎样把一个“只在自己机器上有效”的修复，整理成可以 Upstream 的 Linux/mt76 Patch
## 以 D-Link DIR-X3260 / MT7622 为例

> 项目：D-Link DIR-X3260 A1 OpenWrt reverse engineering  
> 驱动：Linux / mt76 / mt7615 / MT7622 WMAC  
> 本文主题：从 downstream production fix 到 upstream candidate  
> 核心原则：**先证明 scope，再讨论 patch 是否漂亮**

---

## 1. Production 已经工作，为什么还不能直接提交？

这是整个 DIR-X3260 项目后期最容易产生误解的问题。

我们已经有一套经过实体设备验证的 production firmware：

```text
2.4 GHz      PASS
5 GHz        PASS
Internet     PASS
cold boot    PASS
5G LED       PASS
reboot loop  NONE
```

MT7622 2.4 GHz 的 downstream production path 也已经稳定。

直觉上似乎只剩：

```text
git format-patch
        ↓
send upstream
```

但 upstream engineering 真正困难的部分恰恰从这里开始。

因为：

```text
“这个修改修好了我的 DIR-X3260”
```

只证明：

```text
DIR-X3260 needs / benefits from this behavior
```

它并没有自动证明：

```text
all MT7622 devices need this behavior
```

更没有证明：

```text
the current patch architecture is appropriate upstream
```

所以 production success 与 upstream readiness 是两个不同的问题。

---

## 2. Downstream fix 允许知道“板子是谁”，generic driver 不应该随便猜

在设备专用 OpenWrt firmware 中，我们天然知道目标就是：

```text
D-Link DIR-X3260 A1
```

因此 downstream patch 即使包含针对这台设备的特殊处理，只要：

- 不破坏其他目标；
- 行为经过验证；
- release scope 明确；

就可以是合理的工程选择。

但 upstream mt76 面对的是整个设备生态。

MT7622 WMAC 并不只存在于 DIR-X3260。

因此一个类似：

```c
if (is_mt7622(dev))
        enable_special_behavior();
```

的修改，实际表达的是：

> **所有使用 MT7622 WMAC 的设备都应该这样做。**

这是一项比“DIR-X3260 这样做能工作”强得多的技术声明。

如果没有证据，就不能这样写。

---

## 3. A1 最初的问题：`is_mt7622()` 太宽了

DIR-X3260 production 中一个重要行为是 WPDMA global configuration 的 BIT30：

```text
MT_WPDMA_GLO_CFG_CLK_GATE_DIS = BIT(30)
```

实体设备已经给出很强的 DIR-X3260 runtime evidence。

因此最初很自然会想到：

```c
if (is_mt7622(dev))
        val |= MT_WPDMA_GLO_CFG_CLK_GATE_DIS;
```

代码很简单。

甚至看起来非常“干净”。

但它的问题不是代码风格。

问题是它的逻辑 scope：

```text
is_mt7622()
```

等价于声称：

```text
MT7622 SoC-wide requirement
```

而我们的证据只到：

```text
DIR-X3260 requirement
```

这两个集合并不相等。

---

## 4. 第一项 upstream 审查：还有哪些 MT7622 board？

我们随后专门审查了 OpenWrt/Linux 中其他 MT7622 WMAC 使用者。

结果很明确：

> DIR-X3260 并不是唯一的 MT7622 WMAC board。

这立即使 SoC-wide patch 变得危险。

因为如果提交：

```c
if (is_mt7622(dev))
```

就意味着这些未经 DIR-X3260 实验验证的设备也会改变 WPDMA behavior。

即使我们的 DIR-X3260 hardware test 是 100% PASS，也不能替这些设备完成验证。

因此第一个 upstream 结论是：

```text
DIR-X3260 runtime proof
        ≠
MT7622-wide proof
```

---

## 5. 第二项审查：OEM driver 能不能证明 BIT30 是 SoC 规则？

如果 D-Link OEM binary 明确显示：

```text
MT7622 WfHifHwInit
    → set WPDMA BIT30
```

那至少可以提供另一条 provenance evidence。

项目早期一度有类似解释。

但后来的专门 OEM provenance audit 重新检查了相关反汇编。

最终必须收紧结论：

> **OEM WfHifHwInit / SetWPDMA 对 BIT30 的 programming 没有被证明。**

我们能够证明 OEM 会访问相关 WPDMA 区域和寄存器路径。

但：

```text
access to register
        ≠
proof that BIT30 is set
```

因此 upstream 文档不能再写：

```text
D-Link OEM sets BIT30, therefore Linux should too.
```

这句话超出了证据。

这也是为什么 reverse engineering 项目必须允许自己推翻早期解释。

---

## 6. 一个重要原则：错误的 provenance 比没有 provenance 更危险

如果不知道 OEM 为什么这样做，我们会保持谨慎。

但如果我们错误地相信：

```text
OEM definitely sets BIT30
```

就很容易进一步推导：

```text
this is the intended MT7622 hardware sequence
```

再进一步：

```text
all MT7622 should do it
```

这样一个未经证明的反汇编解释，就可能最终变成影响整个 SoC family 的 upstream behavior。

因此 upstream preparation 阶段的目标不是“尽量为 patch 找理由”。

而是：

> **主动攻击自己的证据链。**

如果证据经不起攻击，就缩小 claim。

---

## 7. 从 SoC-wide 改成 board-specific

既然已经证明：

```text
DIR-X3260 runtime needs the behavior
```

但没有证明：

```text
all MT7622 need the behavior
```

那么正确的架构自然变成：

```text
DIR-X3260-specific quirk
```

而不是：

```text
MT7622-wide behavior change
```

这就是 A1 从早期设计走向 v3 的核心变化。

---

## 8. 为什么不直接在 DMA hot path 调 `of_machine_is_compatible()`？

最简单的 board-specific 写法可能是：

```c
if (of_machine_is_compatible("dlink,dir-x3260-a1"))
        ...
```

如果每次 DMA setup 时直接检查 machine compatible，功能上也许能实现目标。

但这不是最好的 driver architecture。

因为 DMA code 的职责应该尽量是：

```text
consume device capability / quirk state
```

而不是：

```text
discover which consumer router model is running
```

所以最终 architecture review 选择：

```text
probe/init
    ↓
detect board once
    ↓
cache internal quirk
    ↓
DMA path reads cached quirk
```

这样 board identification 和 DMA mechanism 被分离。

---

## 9. 为什么 `mt7622_wmac_init()` 是自然的检测点

current mt76 的 MT7622 WMAC 本来就有专门初始化路径：

```text
mt7622_wmac_init()
```

而 device registration flow 会在 hardware initialization 前调用它。

因此这个位置天然适合做：

```text
MT7622-specific platform initialization
        +
board-specific quirk discovery
```

最终思路：

```text
mt7622_wmac_init()
        ↓
is this dlink,dir-x3260-a1?
        ↓
cache quirk = true
```

然后：

```text
DMA initialization
        ↓
if cached quirk
        ↓
apply BIT30 behavior
```

这比在 generic DMA code 中直接查询 machine identity 更清楚。

---

## 10. 为什么没有发明一个新的 Device Tree register-bit property

另一个可能方案是 DTS：

```dts
mediatek,some-wpdma-bit30-quirk;
```

或者更加糟糕：

```dts
mediatek,wpdma-glo-cfg-mask = <0x40000000>;
```

这样似乎很“可配置”。

但 Device Tree 不应该变成：

```text
driver implementation register override database
```

尤其当我们真正知道的是：

```text
this board needs a driver quirk
```

而不是：

```text
this is a stable hardware property that firmware/OS interfaces
should describe forever
```

因此最终 A1 architecture：

```text
NO new DT property
```

直接利用已经存在的 root board compatible：

```text
dlink,dir-x3260-a1
```

识别板子。

---

## 11. A1 v3 的最终设计原则

最终 A1 candidate 满足：

```text
SCOPE:
DIR-X3260 ONLY

SoC-wide MT7622:
NO

new DT property:
NO

DMA runtime machine check:
NO

cached internal quirk:
YES
```

逻辑模型：

```text
probe / mt7622_wmac_init
        ↓
board compatible check
        ↓
cache DIR-X3260 quirk
        ↓
normal initialization
        ↓
DMA setup
        ↓
consume cached quirk
        ↓
apply BIT30 behavior only on DIR-X3260
```

这比最初的 `is_mt7622()` patch 多了代码。

但它少了一个更严重的问题：

```text
unsupported generalization
```

---

## 12. Static apply PASS 到底证明了什么？

A1 v3 最终针对当时 current upstream mt76：

```text
be5ce7910521492d4a2e4ce7ee3843680a46c047
```

完成：

```text
APPLY_CHECK=PASS
APPLY=PASS
DIFF_CHECK=PASS
SCOPE_CHECK=PASS
```

这说明：

- patch 能应用到所审查的 upstream tree；
- diff 与预期一致；
- 没有重新变成 SoC-wide；
- 没有引入新 DT property；
- 没有在 DMA path 做 machine check；
- cached quirk architecture 被保留。

但是 static apply **不等于**：

```text
BUILD PASS
```

更不等于：

```text
RUNTIME PASS
```

STAGE113/114 对这一点明确记录：

```text
BUILD_STARTED=NO
RUNTIME_TESTED=NO
```

因此 A1 的准确状态是：

```text
UPSTREAM CANDIDATE
```

不是：

```text
UPSTREAM-READY AND FULLY VALIDATED
```

---

## 13. 为什么 A1 不能被宣传成“DIR-X3260 完整修复”

这是另一个非常重要的 scope boundary。

production 中 MT7622 能稳定工作的最终行为不是只有 BIT30。

还包括：

```text
IOC / WBSYS
RX_PRE_CFG
ownership semantics
```

因此即使 A1 本身是最适合先进入 upstream review 的一项，也必须明确：

```text
A1 alone
    !=
complete DIR-X3260 production fix
```

否则别人可能：

1. 只应用 A1；
2. 2.4 GHz 仍然失败；
3. 得出“A1 没用”的结论。

而真正的问题是：

> A1 从来没有被证明可以单独代表完整 production fix。

---

## 14. A2：为什么 `RX_PRE_CFG` 最终是 HOLD

production 使用：

```text
MT_WPDMA_RX_PRE_CFG = 0x0f7f0000
```

current upstream 也已经存在相同 register/value 的相关逻辑，但用于已有的 MT7615 path。

这很诱人：

```text
既然 upstream 已经有这个值，
只要把 MT7622 加进去就好了。
```

但这仍然需要证明：

```text
all relevant MT7622 paths need it
```

或者至少：

```text
DIR-X3260 can be isolated as a board quirk
and this variable independently matters
```

我们的 production evidence 证明：

```text
DIR-X3260 known-good stack includes RX_PRE_CFG
```

却没有足够强的单变量证据证明：

```text
RX_PRE_CFG alone is independently required
```

也没有证明：

```text
all MT7622 need it
```

因此：

```text
A2 = HOLD
```

这不是说 A2 “错误”。

而是：

```text
evidence insufficient for submission scope
```

---

## 15. IOC：OEM evidence 很强，为什么仍然 HOLD？

production 使用：

```text
infracfg base    = 0x10000000
offset           = 0x320
physical         = 0x10000320
BIT(31)
```

这部分有 DIR-X3260 OEM evidence，也有 production runtime success。

为什么不直接 upstream？

因为当前 downstream implementation 的 scope 仍然过宽。

我们能够说：

```text
DIR-X3260:
this IOC behavior is part of known-good production
```

但还不能说：

```text
generic MT7622:
always perform this IOC programming
```

如果要 upstream，它同样需要重新设计 scope 和 architecture。

因此最终：

```text
IOC = HOLD
```

这里最重要的区别是：

```text
technical validity on tested board
        ≠
upstream scope validity
```

---

## 16. A3：为什么 ownership sequence 最终留在 downstream

production 中恢复出的 raw ownership sequence：

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

DIR-X3260 production runtime 已经验证。

但 current upstream mt76 的 ownership implementation 使用的是自己的 generic macros/trigger semantics。

要把 production raw-bit sequence upstream，就必须回答：

```text
现有 upstream semantics 对哪些设备错误？
为什么？
OEM exact sequence 的适用范围是什么？
是否只有 DIR-X3260？
是否所有 MT7622？
是否会影响其他 mt7615-family devices？
```

现有证据不足以安全回答这些问题。

因此：

```text
A3 = DOWNSTREAM ONLY
```

这比“先提交，让 maintainer 帮忙判断”更负责任。

---

## 17. B1：有时候最好的 patch 是“不再提交”

MT7915 LED investigation 过程中 production tree 还包含：

```text
004-pass-LED-define-to-mt7915-via-ccflags-y.patch
```

它把：

```make
EXTRA_CFLAGS += -DCONFIG_MT76_LEDS
```

改成：

```make
ccflags-y += -DCONFIG_MT76_LEDS
```

upstream preparation 时重新检查 current mt76 后发现：

```text
B1 behavior already upstream
```

因此最终：

```text
B1 = ALREADY UPSTREAM
```

而不是重新发送一个重复 patch。

这说明 upstream preparation 必须以：

```text
current upstream
```

为基准，而不能只看自己的历史 downstream tree。

---

## 18. B2：MT7915 OEM LED 为什么必须独立 review

DIR-X3260 5 GHz LED 的 production behavior 已经实体验证：

```text
logical LED = 1
source      = 26
EXT_ID      = 0x17

MAP
OFF
ON

start_ap replay
```

但这是一个非常明确的 board-specific OEM MCU behavior。

它与 MT7622 2.4 GHz core fix 不是同一个问题。

因此最终分类：

```text
B2 = SEPARATE REVIEW REQUIRED
```

不能为了“形成一个完整系列”就把它塞进 MT7622 patch series。

好的 patch series 不以数量完整为目标。

它以：

```text
one coherent technical argument per change
```

为目标。

---

## 19. 最终分类为什么只有 A1 可以继续

STAGE114 的最终决策：

```text
A1   = UPSTREAM CANDIDATE
A2   = HOLD
IOC  = HOLD
A3   = DOWNSTREAM ONLY
B1   = ALREADY UPSTREAM
B2   = SEPARATE REVIEW REQUIRED
```

这张表表面上看像是：

```text
六项工作只剩一项可以提交
```

但实际上它说明 upstream preparation 成功了。

因为它消除了：

- 重复 upstream 的 patch；
- scope 过宽的 patch；
- 缺少独立证据的 patch；
- 会改变 generic semantics 的高风险 patch；
- 与主 series 无关的 board feature。

最后留下的 A1 才有一个相对清楚、诚实、可 review 的 technical claim。

---

## 20. Upstream patch 的 commit message 应该证明什么？

一个好的 commit message 不应该写成：

```text
Fix MT7622 Wi-Fi.
```

因为这比证据强太多。

更准确的论证应该围绕：

```text
D-Link DIR-X3260 A1
        ↓
observed WPDMA/ownership failure
        ↓
tested board-specific requirement
        ↓
avoid changing all MT7622
        ↓
cache a DIR-X3260 quirk during init
        ↓
consume it in DMA setup
```

也就是说 commit message 应回答：

1. **谁坏了？**
2. **表现是什么？**
3. **这个修改在什么硬件上被证明？**
4. **为什么不能做成 generic behavior？**
5. **为什么选择这个 architecture？**
6. **哪些东西这个 patch 明确没有解决？**

这比一句“fix driver own failed”更容易被 maintainer 正确 review。

---

## 21. Upstream engineering 的一个反直觉事实

很多人会觉得：

```text
patch 越 generic
    → 越 upstream-friendly
```

实际上并不总是如此。

如果 evidence 只支持一个 board，那么：

```text
well-scoped board quirk
```

往往比：

```text
elegant but unproven SoC-wide change
```

更适合 review。

Upstream 追求的不只是少几个 `if`。

更重要的是：

```text
correct abstraction
correct scope
no regressions
maintainable rationale
```

---

## 22. “我测试过”到底证明多少？

实体测试是极高价值证据，但它也有 scope。

例如我们测试：

```text
DIR-X3260 A1
```

完整通过。

它能够强力证明：

```text
this exact production stack works on this tested hardware
```

但不能自动证明：

```text
all MT7622 boards behave identically
```

同样，OEM binary 来自 DIR-X3260，也主要证明：

```text
D-Link's implementation for this product
```

不是：

```text
MediaTek's universal rule for every MT7622 design
```

因此 evidence 的质量和 evidence 的覆盖范围是两个不同维度。

---

## 23. 一个实用的 Upstream Evidence Ladder

这个项目最后可以总结出一套 evidence ladder。

### Level 1 — symptom

```text
driver own failed
```

只能证明发生了失败。

### Level 2 — diagnostic correlation

```text
changing BIT30 changes behavior
```

说明值得继续研究。

### Level 3 — controlled runtime evidence

```text
known-good / known-bad A/B
```

开始建立因果关系。

### Level 4 — OEM provenance

```text
binary disassembly / call path / register semantics
```

解释厂商行为。

### Level 5 — production validation

```text
boot
Wi-Fi
Internet
cold boot
repeated lifecycle
```

证明完整产品 stack。

### Level 6 — cross-device scope evidence

```text
other boards / datasheets / upstream architecture
```

决定能否 generalize。

DIR-X3260 A1 的 BIT30 已经拥有很强的 board-level evidence。

但缺少 Level 6 的 MT7622-wide proof。

所以正确结果就是：

```text
board-specific candidate
```

而不是：

```text
SoC-wide fix
```

---

## 24. 什么情况下应该 HOLD，而不是继续“优化 patch”？

如果问题是代码风格：

```text
rename helper
move function
split struct
```

继续优化有意义。

但如果问题是：

```text
we do not know whether this applies to all devices
```

再怎么重构代码都不会制造新的硬件证据。

这时应该：

```text
HOLD
```

而不是：

```text
keep polishing until it looks upstreamable
```

A2 和 IOC 就属于这种情况。

HOLD 是技术状态，不是失败。

---

## 25. 什么情况下应该 DOWNSTREAM ONLY？

A3 是很好的例子。

我们知道 production sequence：

```text
works
```

但 upstream 已经有不同 generic ownership semantics。

如果没有足够证据证明 upstream generic path 错误，就不应该用一台设备的 raw-bit sequence 改写它。

所以：

```text
DOWNSTREAM ONLY
```

意味着：

> 这个行为对当前产品是有价值、经过验证的，但现阶段没有足够依据把维护成本和 regression risk 转移给整个 upstream ecosystem。

这是一个完全合理的最终状态。

---

## 26. Production branch 为什么必须冻结

upstream preparation 的另一个纪律是：

> **不要为了 upstream 漂亮，继续修改已经通过硬件验证的 production branch。**

DIR-X3260 production 已经冻结在：

```text
branch:
dirx3260-production-final

commit:
b6edaaf4e5df8596a742975741428217d24d11e3

tree:
40e472af26dbf5679a45da21f053ca0eccf3ec2a
```

upstream architecture exploration 与 production release 是两个不同轨道。

原因很简单：

如果为了 upstream 重构 production source：

```text
source changed
    ↓
previous runtime validation no longer exactly applies
    ↓
must rebuild
    ↓
must re-run full hardware gate
```

没有必要为了代码美观主动破坏已经建立的 release identity。

---

## 27. Upstream Candidate 与 Production Final 可以同时成立

最终项目状态并不矛盾：

```text
Production:
KNOWN-GOOD PRODUCTION FINAL
```

同时：

```text
Upstream:
A1 CANDIDATE
A2 HOLD
IOC HOLD
A3 DOWNSTREAM ONLY
```

Production 回答：

> **这台 DIR-X3260 怎样可靠工作？**

Upstream 回答：

> **哪些知识已经足够成熟，可以安全加入共享 driver？**

这本来就是两个不同问题。

---

## 28. Current upstream 必须重新检查

在准备 patch 时，我们还发现一个很典型的问题：

历史 production tree 中的 B1：

```text
EXTRA_CFLAGS
    →
ccflags-y
```

在 current upstream 已经存在等价行为。

这说明 patch preparation 不能只做：

```text
take old downstream diff
    ↓
rebase
```

而应该：

```text
fetch current upstream
        ↓
read current implementation
        ↓
classify every downstream delta again
        ↓
drop what upstream already solved
        ↓
redesign what no longer fits
```

Upstream 是移动的目标。

---

## 29. 最终工作流

如果把 DIR-X3260 的经验抽象成通用流程，可以写成：

```text
1. Establish known-good production behavior
        ↓
2. Freeze production source/binary identity
        ↓
3. Inventory every downstream semantic delta
        ↓
4. Fetch current upstream
        ↓
5. Check whether each delta already exists
        ↓
6. Separate board evidence from SoC evidence
        ↓
7. Audit OEM provenance again
        ↓
8. Attack over-broad assumptions
        ↓
9. Redesign scope
        ↓
10. Avoid unnecessary DT ABI
        ↓
11. Cache quirks at natural init points
        ↓
12. Keep generic paths generic
        ↓
13. Static apply / diff / scope check
        ↓
14. State clearly what has NOT been tested
        ↓
15. Submit only the evidence-supported subset
```

这比：

```text
make patch smaller
```

重要得多。

---

## 30. 最终结论

DIR-X3260 upstream preparation 最有价值的结果，不是最终生成了一个 A1 v3 patch。

真正的结果是我们学会了把下面三句话分开：

```text
It works on my router.
```

```text
I understand why it works on my router.
```

```text
I have enough evidence to change the shared driver.
```

第一句话需要实体测试。

第二句话需要 reverse engineering、source analysis 和 controlled experiments。

第三句话还需要：

```text
scope evidence
upstream architecture review
regression reasoning
current-tree verification
```

DIR-X3260 最终没有把所有 production modifications 都包装成 upstream patch。

相反：

```text
A1   → candidate
A2   → hold
IOC  → hold
A3   → downstream only
B1   → already upstream
B2   → separate review
```

这不是 upstream preparation 做得不够。

恰恰相反：

> **知道什么应该提交，以及更重要的——知道什么现在不应该提交——才是把一个“只在自己机器上有效”的修复变成真正 upstream engineering 的分界线。**

---

## 附：A1 v3 最终身份

```text
File:
0001-wifi-mt76-mt7615-add-DIR-X3260-WPDMA-clock-quirk-v3.patch

SHA256:
07af4f9d410fa143cafa984795fbc9b42705f645fe5b9dae20b5fc7eac3f5ca5
```

验证基准：

```text
upstream mt76:
be5ce7910521492d4a2e4ce7ee3843680a46c047
```

静态验证：

```text
A1_APPLY_CHECK=PASS
A1_APPLY=PASS
A1_DIFF_CHECK=PASS
A1_SCOPE_CHECK=PASS

A1_SCOPE=DIR-X3260_ONLY
A1_SOC_WIDE_MT7622=NO
A1_NEW_DT_PROPERTY=NO
A1_DMA_MACHINE_CHECK=NO
A1_CACHED_QUIRK=YES
```

验证边界：

```text
BUILD_STARTED=NO
RUNTIME_TESTED=NO

A1_ALONE_DOES_NOT_REPRESENT_COMPLETE_DIRX3260_PRODUCTION_FIX=YES
```

这组边界与 patch 本身同样重要。
