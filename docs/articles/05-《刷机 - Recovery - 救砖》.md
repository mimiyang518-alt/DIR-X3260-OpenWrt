# D-Link DIR-X3260 A1 OpenWrt 完整刷机、Recovery 与救砖指南

> 本文面向已经决定在 D-Link DIR-X3260 A1 上使用本项目 OpenWrt 固件的用户。重点不是“怎样把一个文件写进去”，而是怎样建立一条**可恢复、可验证、不会因为一次坏构建就失去设备**的刷机流程。

## 1. 先理解两个完全不同的网络环境

DIR-X3260 的 D-Link Recovery 环境和刷入 OpenWrt 后的正常系统不是同一个网络。

### D-Link Recovery

电脑网卡手工设置到：

```text
192.168.0.xxx
```

例如：

```text
IP:      192.168.0.10
Mask:    255.255.255.0
Gateway: 可留空
```

### OpenWrt 正常系统

进入 OpenWrt 后，管理网络位于：

```text
192.168.1.xxx
```

如果电脑没有自动取得地址，调试时可以暂时手工设置：

```text
IP:      192.168.1.10
Mask:    255.255.255.0
```

测试完成后，建议把电脑 IPv4 恢复为 **DHCP / 自动获取**。

这两个网段不要混淆。很多“Recovery 页面打不开”或“刷完以后 OpenWrt 访问不了”的问题，本质只是电脑仍停留在错误的静态网段。

---

## 2. 为什么 Recovery 比刷机命令更重要

逆向工程期间，我们遇到过真正的坏固件。

其中一个明确的 known-bad 构建：

```text
DIRX3260-A1-STAGE55-OEM-EXACT-5G-LED-SYSUPGRADE.bin
```

刷入后会出现立即重复 reboot loop。

这件事说明：

> 在修改内核、无线驱动、MCU command、DMA 或板级初始化路径之前，必须先确认 Recovery 路径确实可用。

不要把 Recovery 当作“出事以后再研究”的功能。

它应该是实验开始前就验证好的安全设施。

---

## 3. 建立 Known-Good Baseline

在测试任何新固件之前，至少保留一份已经经过实体设备验证的 known-good image。

本项目最终锁定的 Production Final sysupgrade：

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin

SHA256:
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

Recovery：

```text
DIRX3260-A1-STAGE72-FINAL-PRODUCTION-RECOVERY.bin

SHA256:
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

这里最重要的不是文件名，而是：

```text
文件
+ SHA256
+ 来源
+ 实体设备验证记录
```

四者必须对应。

---

## 4. 刷写前先校验文件

在 Linux / WSL 中：

```bash
sha256sum DIRX3260-A1-STAGE72-FINAL-PRODUCTION-SYSUPGRADE.bin
```

预期：

```text
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

不要因为文件名看起来正确就直接刷。

开发过程中同名复制、旧目录、浏览器重复下载、临时构建都可能让“看起来是那个文件”的东西实际上不是那个二进制。

---

## 5. 在 OpenWrt 中测试 sysupgrade image

把固件复制到路由器，例如：

```text
/tmp/firmware.bin
```

先做 image test：

```bash
sysupgrade -T /tmp/firmware.bin
```

只有检查通过后才继续。

正式升级：

```bash
sysupgrade /tmp/firmware.bin
```

如果测试需要明确清除旧配置：

```bash
sysupgrade -n /tmp/firmware.bin
```

`-n` 会改变配置保留行为，因此不要机械地在每一次升级中使用。是否保留配置应当是测试设计的一部分。

---

## 6. Recovery → OpenWrt 的实际恢复思路

本项目曾经实际使用过这样的恢复路径：

```text
坏构建 / 异常系统
        ↓
D-Link Recovery
        ↓
刷入已知可工作的 recovery
        ↓
进入 OpenWrt
        ↓
上传 known-good sysupgrade 到 /tmp
        ↓
sysupgrade -T
        ↓
sysupgrade -n（需要干净配置时）
        ↓
重新启动
        ↓
完整硬件验证
```

这比在一个已经不稳定的系统中不断尝试“修回来”可靠得多。

---

## 7. “能启动”绝对不等于刷机成功

本项目最后的实体设备 gate 包括：

1. 无 reboot loop。
2. `OpenWRT` 2.4 GHz SSID 出现并可连接。
3. `OpenWRT-5G` 5 GHz SSID 出现并可连接。
4. Internet 正常。
5. 系统稳定后，5 GHz 真正工作时 LED 点亮。
6. LuCI disable 5 GHz 后 SSID 消失，LED 完全熄灭。
7. LuCI enable 5 GHz 后等待 SSID 真正恢复广播，LED 再点亮。
8. 客户端重新连接 5 GHz，数据和 Internet 正常。
9. 再重复一次 disable → OFF → enable → ON。
10. 冷启动后重新验证两条无线、Internet 和 LED 生命周期。

只有这些都通过，我们才把对应 binary 称为：

```text
KNOWN-GOOD PRODUCTION FINAL
```

---

## 8. 为什么 Internet 必须单独测试

开发期间曾经出现过一个很有价值的回归：

```text
2.4 GHz 看起来正常
5 GHz 看起来正常
LED 也可能正常
但 Internet 已经坏了
```

因此验证无线驱动时，不能只看：

```text
iw dev
SSID
LED
```

还必须验证真实数据路径。

一个驱动修改即使解决了目标问题，也可能通过其他路径造成网络回归。

---

## 9. 5 GHz 启动需要等待真实状态

MT7915 5 GHz 并不是“进程起来了”就等于 AP 已经真正广播。

因此测试 LED 时必须以：

```text
OpenWRT-5G 真正可见
客户端真正可连接
数据真正可传输
```

作为状态依据。

最终实现也遵循这一原则：LED 生命周期跟随真实 AP/SSID 状态，而不是简单地在 boot 时强制点亮。

---

## 10. Reboot Loop 时不要反复等待

如果一个新构建在刷入后立即进入重复重启，而 known-good 固件此前稳定工作，就应优先考虑：

```text
停止继续等待
→ 进入 Recovery
→ 回滚 known-good
→ 恢复可工作的测试环境
→ 再分析新构建
```

不要把“也许再等几分钟会好”变成测试方法。

对于 boot loop，保持设备可恢复比保留现场更重要；现场证据应尽可能在构建前、串口、日志或源码层面准备。

---

## 11. Production Final 为什么不再随便重编译替换

最终固件经过实体硬件验证以后，我们冻结的是**具体二进制**，而不是一句“这份源码应该能重新编出来”。

源码当然可以重建。

但新的编译时间、toolchain、package revision、feeds 或环境状态都可能产生不同 binary。

因此：

> 新 rebuild 不能自动继承旧 binary 的 Production Final 身份。

如果要替换正式发布的二进制，就应重新走完整 hardware gate。

---

## 12. 最小安全流程

如果只记住一套流程，请记住：

```text
确认 Recovery 可用
        ↓
保存 known-good recovery + sysupgrade
        ↓
记录 SHA256
        ↓
测试新 image：sysupgrade -T
        ↓
刷入
        ↓
验证 boot
        ↓
验证 2.4G
        ↓
验证 5G
        ↓
验证 Internet
        ↓
验证 LED / LuCI lifecycle
        ↓
冷启动再验证
        ↓
才允许称为 known-good
```

## 结语

DIR-X3260 项目最终能安全走到 Production Final，并不是因为每个实验都成功。

恰恰相反，真正重要的是：**失败以后始终有一条确定的路可以回到已知正确状态。**

Recovery、Known-Good Baseline、SHA256、实体设备 gate 和回滚纪律，本质上属于同一个系统。

它们共同回答一个问题：

> 当下一次实验失败时，我们怎样证明自己仍然知道“正确状态”在哪里？
