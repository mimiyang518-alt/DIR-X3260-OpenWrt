# 逆向 D-Link DIR-X3260 OEM 驱动：如何找回 MT7915 5G LED 的 MCU 控制协议

> 设备：D-Link DIR-X3260 A1  
> SoC：MediaTek MT7622  
> 5 GHz radio：MediaTek MT7915  
> 状态：KNOWN-GOOD PRODUCTION FINAL，已完成实体设备反复验证

---

## 1. 一个看似很小、实际上完全不是 GPIO 的问题

DIR-X3260 的 5 GHz 无线本身很早就已经能够正常工作：

```text
OpenWRT-5G visible
client association works
data traffic works
Internet works
```

但前面板 5 GHz LED 的行为一直不对。

这件事一开始很容易被理解成：

```text
找到 LED
    ↓
找到 GPIO
    ↓
写 0 / 1
    ↓
完成
```

Linux 也确实暴露过 `white:wlan-5ghz` 一类 LED/GPIO 信息。

问题是：这些信息能帮助我们找到“灯”，却不能重现 D-Link OEM 的控制方式。

最后证明，真正的路径是：

```text
OpenWrt / mt7915
        ↓
MT7915 MCU command
        ↓
MT7915 firmware LED engine
        ↓
logical LED mapping
        ↓
source 26
        ↓
front-panel 5 GHz LED
```

所以这不是一个普通的“GPIO 灯不亮”问题。

---

## 2. GPIO 路线为什么会把人带偏

早期调查尝试过：

- Linux LED trigger；
- `phy1radio` / netdev 风格控制；
- gpio-leds；
- GPIO 86 方向；
- legacy GPIO ioctl；
- GPIO v2；
- 强制 ON/OFF；
- DTS LED 描述。

这些实验不是毫无价值。它们帮助确认了：

1. 前面板确实有对应的 5 GHz 指示灯；
2. Linux 可以看到与 LED 相关的板级描述；
3. 仅仅操作 Linux 侧 GPIO 表象，无法得到 OEM 的完整生命周期。

真正需要解释的不是：

```text
“哪个 Linux GPIO 能让它亮？”
```

而是：

```text
“D-Link OEM 驱动到底把这个 LED 交给谁控制？”
```

答案最终指向 MT7915 MCU LED engine。

这与 upstream MT7915 本身具有 LED register/mux 支持并不矛盾；公开 mt76/Linux 代码也包含 MT7915 LED mux、LED registers 和 GPIO 26 mux 定义。DIR-X3260 的特殊之处在于，我们需要重现 OEM 在这块板上使用的 MCU command/lifecycle，而不是假定普通 host-side GPIO toggle 与 OEM 行为等价。

---

## 3. 从 OEM 符号开始追踪

OEM MT7915 驱动中出现了一组高价值符号：

```text
RTMPInitLEDMode
RTMPStartLEDMode
RTMPExitLEDMode
RTMPSetLEDStatus
LEDControlTimer
wps_led_control
mt7915_wps_led_control
rtmp_control_led_cmd
AndesLedEnhanceOP
```

这使调查方向从：

```text
Linux LED class / GPIO
```

转向：

```text
OEM LED logic
    ↓
Andes MCU command construction
    ↓
firmware LED engine
```

这里最重要的原则是：

> **不能看到一个 operation number 就猜它是 ON 或 OFF。**

必须同时证明：

- call site；
- argument；
- payload constructor；
- exact packet bytes；
- 实机灯状态。

否则一个“灯刚好亮了”的实验，很容易被错误冻结成协议定义。

---

## 4. 第一个真正确定的结果：LED index 1，source 26

逆向最终恢复出 DIR-X3260 5 GHz LED 的 OEM mapping：

```text
logical LED index = 1
source / GPIO     = 26
flag              = 0
```

对应 OEM 语义：

```text
AndesLedGpioMap(adapter, 1, 26, 0)
```

OpenWrt board description 最终也保留：

```dts
led-sources = <26>;
```

但这里必须特别强调：

> `26` 是 MT7915 LED-engine mapping 中恢复出的 source。它不是“普通 host GPIO 26 toggle 就等价于 OEM 实现”的证明。

这是整个 LED 调查中最容易混淆的两个层次。

---

## 5. MAP packet 是怎么确定的

恢复出的 exact MAP payload：

```text
02 05 01 00 00 34 00 00
```

它对应：

```text
logical LED = 1
source      = 26
flag        = 0
```

其中最重要的结论不是某个字节长什么样，而是：

**这是 MAP command，不是 ON command，也不是 OFF command。**

早期如果把它错误理解成“发了这个包灯亮，所以它就是 ON”，后面的生命周期一定会出问题。

最终 production 把 source map 作为初始化语义保留下来：

```text
mt7915_mcu_dirx3260_led_oem(dev, 0)
```

也就是说，在谈 ON/OFF 之前，先建立：

```text
logical LED 1 → source 26
```

---

## 6. MCU command identity：EXT_ID 0x17

OEM 路径最终对应到 MT7915 MCU extended command。

production backend 使用：

```c
MCU_CMD(EXT_CID) |
FIELD_PREP(__MCU_CMD_FIELD_EXT_ID, 0x17)
```

即：

```text
EXT_ID = 0x17
```

公开 mt76 的 MCU command infrastructure 本身就有 EXT_CID/extended-command 编码机制；DIR-X3260 的逆向工作确定的是 OEM LED command 在这套机制中的具体 ID、payload 和板级行为。

到这里，问题已经从“GPIO 怎么写”变成：

```text
EXT_CID 0x17
        +
8-byte LED payload
        +
logical LED mapping
        +
runtime lifecycle
```

---

## 7. 真正的 OFF packet

最终通过 constructor 分析和实体设备验证确定：

```text
02 01 01 00 00 00 00 00
```

其意义是：

```text
CONTROL = 0
LED     = 1
state   = OFF
```

在最终 helper 的 operation mapping 中：

```text
op 1
    ↓
CONTROL=0
    ↓
OFF
```

这不是通过 operation number 猜出来的，而是 runtime verified。

---

## 8. 真正的 ON packet

对应的 ON payload：

```text
02 01 01 01 00 00 00 00
```

含义：

```text
CONTROL = 1
LED     = 1
state   = ON
```

最终：

```text
op 3
    ↓
CONTROL=1
    ↓
ON
```

因此完整的 production command family 是：

```text
MAP:
02 05 01 00 00 34 00 00

OFF / CONTROL=0:
02 01 01 00 00 00 00 00

ON / CONTROL=1:
02 01 01 01 00 00 00 00
```

这三个包必须区分：

```text
MAP ≠ ON
MAP ≠ OFF
CONTROL=0 = OFF
CONTROL=1 = ON
```

---

## 9. 为什么“能手工点亮”仍然不算完成

即使已经知道 ON packet，问题仍然没有结束。

一个路由器前面板 Wi-Fi LED 真正需要表达的是**无线状态**，不是“驱动曾经发过一次 ON”。

我们最终要求的是：

```text
5G 真正可用
    → LED ON

LuCI disable 5G
    → SSID 消失
    → LED OFF

LuCI enable 5G
    → AP 真正恢复
    → SSID 出现
    → LED ON
```

这意味着 MCU protocol 只是第一半。

第二半是：**什么时候发送这些 command。**

---

## 10. Linux LED class 仍然有价值

最终 production 并没有把 Linux LED framework 全部绕开。

相反，它保留正常 brightness lifecycle，但在 DIR-X3260 上把 brightness 转换为 OEM MCU command：

```c
mt7915_mcu_dirx3260_led_oem(dev, brightness ? 3 : 1);
```

也就是：

```text
brightness == 0
    → op 1
    → CONTROL=0
    → OFF

brightness != 0
    → op 3
    → CONTROL=1
    → ON
```

并且这个 backend 被限制到：

```text
dlink,dir-x3260-a1
```

其他 MT7915 设备继续走 generic LED path。

这是一种很重要的工程折中：

```text
Linux LED lifecycle
        +
DIR-X3260-specific MCU backend
```

而不是：

```text
完全抛弃 Linux LED framework
```

---

## 11. 最后一个关键突破：`start_ap` replay

做到 brightness → MCU ON/OFF 后，仍然有一个 startup timing 问题。

如果 ON 在初始化过早阶段发送：

```text
driver initialized
        ≠
5 GHz AP actually operational
```

所以最终 production 在 MCU 初始化后安装 source map，而在 AP/BSS/STA setup 成功之后重新发送：

```c
mt7915_mcu_dirx3260_led_oem(dev, 3);
```

也就是：

```text
CONTROL=1 / ON
```

最终启动顺序变成：

```text
boot
  ↓
MT7915 MCU starts
  ↓
source 26 map installed
  ↓
AP/BSS/STA startup succeeds
  ↓
OpenWRT-5G becomes operational
  ↓
CONTROL=1 replay
  ↓
5G LED ON
```

这一步解决的不是“如何点亮”，而是：

> **如何让 LED 的可见状态代表真正已经启动成功的 5 GHz AP。**

---

## 12. LuCI disable / enable 最终生命周期

最终实体设备上的行为：

```text
OpenWRT-5G stable
        ↓
LED ON
```

关闭 5 GHz：

```text
LuCI disable
        ↓
OpenWRT-5G disappears
        ↓
brightness OFF
        ↓
CONTROL=0
        ↓
LED OFF
```

重新开启：

```text
LuCI enable
        ↓
MT7915/AP startup
        ↓
OpenWRT-5G actually returns
        ↓
start_ap replay
        ↓
CONTROL=1
        ↓
LED ON
```

而且这个 disable → OFF → enable → ON 流程进行了重复验证，不是一次偶然现象。

---

## 13. STAGE55：最有价值的一次失败

这个项目里必须保留一个反例：

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

它能够正常 build，导出的 image/hash 也没有问题。

但刷入实体设备后：

```text
immediate repeated reboot loop
```

因此它被正式分类为：

```text
KNOWN BAD
```

当时使用的是另一组候选 state payload：

```text
00 00 00 00 02 00 00 00
00 00 00 00 02 00 00 01
```

这次失败留下的工程结论非常重要：

```text
compile success
        ≠
runtime safety

valid firmware image
        ≠
runtime safety

matching SHA256
        ≠
runtime safety
```

最终只能通过 D-Link recovery 回到 known-good baseline，再继续调查。

因此，STAGE55 不是应该从历史中删除的“错误”。

它是证明**为什么协议逆向必须由实体设备验证闭环**的关键证据。

---

## 14. STAGE56–58：从 reboot loop 回到可验证路径

STAGE55 之后没有继续在危险 payload 上叠加修改，而是退回 boot-safe baseline。

随后逐步确定：

```text
op 1 = CONTROL=0 = OFF
op 3 = CONTROL=1 = ON
```

再完成：

```text
brightness dispatch
        ↓
board gating
        ↓
source map after MCU init
        ↓
start_ap replay
```

STAGE58 成为第一个强验证的 known-good LED baseline：

```text
2.4G             PASS
5G               PASS
Internet         PASS
LED boot ON      PASS
disable → OFF    PASS
enable → ON      PASS
client reconnect PASS
reboot loop      NONE
```

这时才能说：

**我们恢复的不是一个“会亮的 packet”，而是 OEM 风格的可用生命周期。**

---

## 15. Production consolidation

已验证行为最初分散在历史 patches `005`–`009`。

最终 consolidation 把它们整理为：

```text
package/kernel/mt76/patches/
005-dirx3260-mt7915-5g-led-production-final.patch
```

SHA256：

```text
f6f5b1b0dfef24f6c6a78e41f8eff1609d119e062e7a922540aefe2f5260e064
```

consolidation audit 确认关键执行语义保持一致，同时移除了生产环境不需要的手工 debugfs LED control。

也就是说：

```text
reverse-engineering instrumentation
        → preserved as evidence

manual diagnostic interface
        → removed from production

verified MCU semantics
        → preserved
```

---

## 16. 最终 production 语义

如果未来需要审查、重构或重新实现，至少必须保持：

```text
BOARD:
    dlink,dir-x3260-a1

DTS:
    led-sources = <26>

OEM:
    logical LED = 1
    source      = 26

MCU:
    EXT_ID = 0x17
```

以及：

```text
MAP:
02 05 01 00 00 34 00 00

OFF:
02 01 01 00 00 00 00 00

ON:
02 01 01 01 00 00 00 00
```

运行时：

```text
brightness == 0
    → OFF

brightness != 0
    → ON

after MCU init
    → install source map

after successful AP startup
    → replay ON
```

这才是 production equivalence 的最低语义要求。

---

## 17. 最终实体设备验收

最终 production firmware 的 LED gate 包含：

```text
cold boot
    → no reboot loop

OpenWRT
    → visible/connect/data PASS

OpenWRT-5G
    → visible/connect/data PASS

Internet
    → PASS

5G stable
    → LED ON

LuCI disable 5G
    → SSID disappears
    → LED completely OFF

LuCI enable 5G
    → wait until OpenWRT-5G really broadcasts
    → LED ON

client reconnect
    → PASS

disable → OFF → enable → ON
    → repeated PASS
```

这比“LED 能亮”严格得多。

最终验证的是：

> **LED 与真实 5 GHz AP 生命周期一致。**

---

## 18. 为什么公开的 MT7915 LED 支持仍然不足以直接回答这个问题

公开 Linux/mt76 代码并不是完全没有 MT7915 LED 支持。

它包含 LED register、GPIO mux 和 LED initialization；公开历史 patch 也明确讨论过 MT7915 LED support 和 GPIO 26 mux。

但这些公开信息不能自动告诉我们：

```text
DIR-X3260 logical LED = 1
DIR-X3260 source      = 26
OEM EXT_ID            = 0x17

MAP exact payload
OFF exact payload
ON exact payload

以及：
何时 map
何时 OFF
何时 replay ON
```

这些板级语义来自 OEM binary reverse engineering 与实体硬件验证。

所以这个项目真正补上的不是“MT7915 有 LED 功能”这一常识，而是：

> **DIR-X3260 如何使用 MT7915 LED engine。**

---

## 19. 这次逆向最值得保留的方法论

### 1. “能找到 GPIO”不代表“GPIO 就是 owner”

Linux 看到的板级表示，不一定等于 OEM 实际 runtime control path。

### 2. MAP packet 与 state packet 必须分开

这是 LED 协议逆向中最危险的误判之一。

### 3. operation number 不能代替 payload analysis

必须把 constructor、call site、packet bytes 和 hardware result 对起来。

### 4. 手工 ON 成功不代表 lifecycle 完成

真正产品行为还包括 disable、enable、AP startup 和 cold boot。

### 5. failed firmware 是证据

STAGE55 的 reboot loop 明确告诉我们：静态正确性不能替代硬件安全验证。

### 6. production consolidation 必须比较 effective semantics

不是简单把历史 patches 合并后“能编译”就结束。

---

## 20. 最终结论

DIR-X3260 的 5 GHz LED 最终不是通过寻找一个更合适的 Linux GPIO trigger 修好的。

真正的突破是发现：

```text
front-panel LED
        ↑
source 26
        ↑
MT7915 LED engine
        ↑
EXT_CID 0x17
        ↑
OEM MAP / CONTROL command
        ↑
mt7915 driver lifecycle
```

最终恢复出的协议核心：

```text
LED index = 1
source    = 26
EXT_ID    = 0x17

MAP = 02 05 01 00 00 34 00 00
OFF = 02 01 01 00 00 00 00 00
ON  = 02 01 01 01 00 00 00 00
```

而最后决定产品体验的，是把协议和生命周期结合起来：

```text
MCU ready
    → MAP

5 GHz AP really ready
    → ON

5 GHz disabled
    → OFF

5 GHz re-enabled and really ready
    → ON
```

所以这项工作的最终成果，不是“让一颗灯亮起来”。

而是：

> **从 OEM binary 中恢复 DIR-X3260 的 MT7915 LED MCU protocol，并把它重新接回 OpenWrt 的真实 5 GHz AP 生命周期。**

---

## 证据边界

本文只陈述已经由 OEM 分析、production effective source 和实体设备测试支持的 DIR-X3260 行为。

它不意味着：

- 所有 MT7915 板都使用同一组 OEM payload；
- source 26 等价于普通 host GPIO 26 toggle；
- MAP packet 可以当作 ON/OFF packet；
- 一个能 build 的 LED patch 就具有 runtime safety；
- DIR-X3260 的 MCU LED quirk 应未经单独 review 就推广到 generic mt7915。

这些边界也是这次逆向结果的一部分。
