# 一次 Reboot Loop 教会我们的事：从 Known-Good Baseline 到 Production Gate

> **能编译 ≠ 能启动。  
> 能启动 ≠ 网络正常。  
> 网络正常 ≠ Production Final。**

DIR-X3260 项目中最有价值的经验之一，不是某个寄存器值，也不是某条 MCU command。

而是我们真的刷出过 reboot loop，真的遇到过 Internet regression，也真的从这些失败中建立了一套后来能够保护整个项目的工程流程。

---

## 1. 为什么“Build PASS”是最弱的一层证据

一个 kernel/driver patch 能：

```text
apply
compile
link
produce sysupgrade.bin
```

只能说明：

```text
它通过了构建系统
```

它不能证明：

```text
设备能启动
Wi-Fi 能工作
Internet 能工作
LED lifecycle 正确
冷启动稳定
```

嵌入式系统最大的危险之一，就是把 build success 当成 hardware success。

---

## 2. STAGE55：一个非常明确的失败

项目曾生成：

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

它的目标是继续逼近 OEM 5 GHz LED behavior。

结果不是“LED 没亮”。

而是：

```text
立即 repeated reboot loop
```

这是一个非常重要的分界点。

因为从这一刻开始，测试方法必须回答两个问题：

```text
这个实验怎样证明成功？
这个实验失败以后怎样恢复？
```

第二个问题与第一个同样重要。

---

## 3. Recovery 不是 emergency trick，而是开发基础设施

坏构建之后，项目通过 D-Link Recovery 回到可工作的环境，再刷回 known-good sysupgrade。

于是 Recovery 的角色发生了变化：

```text
以前：
救砖功能

后来：
每一次高风险实验的前置条件
```

这也是为什么最终刷机文档明确区分：

```text
D-Link Recovery → 192.168.0.xxx
OpenWrt         → 192.168.1.xxx
```

恢复流程必须简单到在设备出问题时不需要临时重新研究。

---

## 4. Known-Good Baseline 必须是 binary，不是记忆

开发中很容易说：

```text
“昨天那个版本是好的。”
```

但一天可能构建几十个 image。

真正的 known-good baseline 应至少记录：

```text
filename
size
SHA256
source provenance
runtime test result
```

最终 Production Final sysupgrade：

```text
Size:
10752278

SHA256:
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

Recovery：

```text
Size:
21234012

SHA256:
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

这才是可以在几个月以后仍然回答“哪个版本是真的最终版”的记录。

---

## 5. Internet regression：修好目标功能仍然可能失败

项目后期还发生过另一个典型事件：

某些新的 cleanup / LED 修改附近，设备看起来并没有明显 boot failure，但：

```text
Internet 不通了
```

这比 reboot loop 更容易误判。

因为你可能看到：

```text
LuCI 能开
SSID 能看到
客户端能连接
LED 也有反应
```

然后认为新构建“基本成功”。

实际上，对于路由器：

```text
Internet / forwarding / real data path
```

本身就是核心产品功能。

因此 Production Gate 后来明确把 Internet 作为独立项目。

---

## 6. 一次成功不够

最终 5 GHz LED 验证并不是：

```text
enable
→ LED 亮
→ PASS
```

而是：

```text
启动
→ 5G 真正广播
→ LED ON

LuCI disable 5G
→ SSID 消失
→ LED OFF

LuCI enable 5G
→ 等 SSID 真正恢复
→ LED ON

客户端重新连接
→ data PASS

再重复：
disable → OFF → enable → ON
```

为什么重复？

因为很多 lifecycle bug 第一次状态转换没有问题，第二次才暴露：

```text
stale state
duplicate initialization
lost MCU state
missing replay
race
```

---

## 7. 冷启动和软件重启不是同一件事

最终还必须做：

```text
cold power cycle
```

因为无线芯片、MCU、DMA、bootloader 和电源状态可能在 warm reboot 与真正断电后表现不同。

Production Final 的 cold-boot gate 再次验证：

```text
no reboot loop
2.4G
5G
Internet
5G LED after real activation
```

只有热启动成功，不足以冻结硬件固件。

---

## 8. STAGE72 → STAGE73 → STAGE74 的意义

最终流程可以概括成三层：

### STAGE72 — Candidate binary

产生最终候选 image。

### STAGE73 — Provenance

证明：

```text
这个 binary
```

确实对应：

```text
这个 source / patch stack / build output
```

### STAGE74 — Physical runtime validation

在真实 DIR-X3260 A1 上完成完整 hardware gate。

这三层共同构成：

```text
binary identity
+ source provenance
+ physical validation
```

少一个都不应该叫 Production Final。

---

## 9. Freeze 是一个工程动作

实体设备全部通过以后，项目没有继续“顺手再清理一点”。

而是冻结：

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

这一步的意义是：

> 从此以后，“最终源码”不再是一个不断移动的工作目录，而是一个不可含糊的 Git identity。

---

## 10. 为什么后来停止 cleanup

项目最后采取了非常保守的 cleanup 策略。

例如：

```text
不 git clean
不 make clean
不 make dirclean
保留 build workspace
保留 reverse-engineering artifacts
```

只删除明确确认的 accidental zero-byte shell artifacts。

原因很简单：

> 磁盘空间很便宜，丢失一个已经成功工作的 build/research environment 很贵。

“目录看起来整洁”不是 release requirement。

“证据和可恢复性仍然存在”才是。

---

## 11. Rebuild 不自动继承 Production Final 身份

假设今天从 frozen commit 再运行一次 build。

即使 source commit 完全一样，也可能有：

```text
feeds changes
toolchain changes
package revision changes
timestamp effects
host environment changes
```

因此新 binary 应被视为：

```text
new candidate
```

而不是：

```text
same Production Final
```

如果它要替换发布 binary，就重新验证。

---

## 12. Upstream preparation 又增加了一层证据纪律

Production Final 之后，我们继续研究如何把修复整理成 upstream candidate。

这时又出现一个重要区别：

```text
在 DIR-X3260 上验证有效
≠
适用于所有 MT7622
```

例如 WPDMA BIT30。

DIR-X3260 runtime evidence 很强。

但重新审计 OEM binary 后，我们没有足够证据声称：

```text
OEM 明确进行了同样的 BIT30 programming
```

也没有证据证明：

```text
所有 MT7622 board 都需要它
```

所以最终 upstream classification 是：

```text
A1   WPDMA BIT30 board quirk → CANDIDATE
A2   RX_PRE_CFG              → HOLD
IOC  infracfg BIT31          → HOLD
A3   ownership raw sequence  → DOWNSTREAM ONLY
B1   mt7915 ccflags-y        → ALREADY UPSTREAM
B2   MT7915 OEM MCU LED      → SEPARATE REVIEW
```

Production evidence 与 upstream scope evidence 是不同的问题。

---

## 13. “不要让结论走得比证据更远”

这句话后来成为整个项目最重要的原则。

它适用于：

```text
binary reverse engineering
register semantics
MCU protocol
runtime testing
release engineering
upstream patch scope
```

如果证据只证明：

```text
DIR-X3260 A1 works
```

就写：

```text
DIR-X3260 A1 works
```

不要自动写成：

```text
MT7622 requires this
```

如果 OEM binary 没有证明某个 BIT30 write，就不要为了让故事更漂亮而说 OEM 也这么做。

---

## 14. 一个可复用的 Production Gate

对于类似 router / embedded Linux 项目，可以使用：

```text
[ ] patch applies
[ ] build passes
[ ] image format test passes
[ ] device boots
[ ] no reboot loop
[ ] all radios initialize
[ ] SSIDs really broadcast
[ ] clients connect
[ ] real traffic passes
[ ] Internet / forwarding passes
[ ] UI lifecycle operations pass
[ ] hardware indicators follow real state
[ ] repeated state transitions pass
[ ] cold boot passes
[ ] binary SHA256 recorded
[ ] source provenance recorded
[ ] known-good rollback available
[ ] release source frozen
[ ] release binary frozen
```

注意顺序。

最前面的：

```text
patch applies
build passes
```

只是整个列表的开始。

---

## 15. 最危险的不是失败，而是“半成功”

Reboot loop 很明显。

真正危险的是：

```text
设备能启动
但某个 radio 不工作

radio 能工作
但 Internet 坏了

Internet 正常
但 disable/enable 第二轮坏了

功能都正常
但你不知道发布 binary 到底来自哪个 source state
```

这些状态都比完全失败更容易被错误地发布。

Production Gate 的意义，就是系统性地消灭“看起来差不多成功”。

---

## 结语

DIR-X3260 最终成为 Production Final，不是因为我们找到了一组神奇寄存器。

而是因为项目逐渐建立了一条完整证据链：

```text
reverse engineering
        ↓
source change
        ↓
build
        ↓
binary identity
        ↓
runtime
        ↓
failure recovery
        ↓
repeated hardware validation
        ↓
cold boot
        ↓
provenance
        ↓
freeze
        ↓
archive + backup
```

STAGE55 的 reboot loop 并不是项目中的污点。

从工程角度看，它反而是一个关键事件：

> 它迫使我们把“修复一个问题”升级成“建立一个可以安全地证明修复成立的系统”。

这套方法最终比任何一个单独 patch 都更值得保留下来。
