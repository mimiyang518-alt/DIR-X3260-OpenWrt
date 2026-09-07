# 如何逆向 D-Link SHRS 固件：从加密容器到可验证的 Firmware Round-Trip

> DIR-X3260 项目最早遇到的问题甚至不是 Wi-Fi，而是：**OEM firmware 到底是什么格式？**
>
> 如果连厂商镜像的封装、加密、摘要和签名关系都没有弄清楚，那么后面的 recovery、factory image、rootfs 分析和 OEM driver reverse engineering 都缺少可靠入口。

## 1. 从“binwalk 看不懂”开始

D-Link DIR-X3260 A1 OEM firmware 并不是一个直接暴露 FIT、kernel 或 SquashFS 的普通镜像。

外层首先存在一个 `SHRS` container。

以项目分析的 OEM image 为例，关键结构包括：

```text
SHRS header
    │
    ├── payload length fields
    ├── salt
    ├── SHA-512 digests
    ├── RSA public key
    ├── signature before
    └── signature post

encrypted payload
    │
    └── AES-128-CBC

footer
```

只有正确处理外层以后，内部 Linux firmware 才真正出现。

---

## 2. Header 不是“几个 magic bytes”

项目最终解析出的 header 长度：

```text
0x6dc
```

其中包括：

```text
payload_length_before
payload_length_post

salt[16]

md_vendor
md_before
md_post

rsa_pub[512]

sig_before[512]
sig_post[512]
```

512-byte RSA key/signature 对应：

```text
4096-bit RSA
```

这里一个重要教训是：

> 固件格式逆向不能只停在“我找到了 payload offset”。

必须继续回答每一个字段参与了什么校验。

---

## 3. AES-128-CBC 只是第一层

加密 payload 使用：

```text
AES-128-CBC
```

并涉及 zero padding。

但知道算法名称仍然不够。

真正需要恢复的是：

```text
key 怎样得到
IV 怎样得到
padding 怎样处理
解密后长度怎样验证
```

项目通过 OEM userspace 工具和镜像字段交叉分析，逐渐把这些关系闭合起来，而不是根据常见 D-Link 格式直接猜测。

---

## 4. OEM rootfs 给出了关键证据

解密后的 rootfs 中存在：

```text
/bin/imgdecrypt
/etc/public.pem
/etc/enk.txt
```

`imgdecrypt` 是 AArch64 binary。

这些文件把问题从：

```text
“这个 header 看起来像 RSA/AES”
```

推进到：

```text
“OEM 自己到底怎样验证和解密这个 image”
```

通过 strings、binary analysis 和实际数据验证，可以把 header 字段与 OEM 实现对应起来。

---

## 5. SHA-512 不止一个

SHRS 中不是简单的“payload SHA”。

项目确认存在多个 digest：

```text
md_vendor
md_before
md_post
```

其中一个关键关系：

```text
md_vendor = SHA512(plaintext || vendor_key)
```

这类关系非常重要，因为它说明：

> 即使成功解密 payload，也不代表已经理解 OEM image 的完整认证模型。

如果重新封装以后摘要关系不对，OEM 路径仍可能拒绝镜像。

---

## 6. 两个 4096-bit RSA signatures

Header 中保存：

```text
sig_before[512]
sig_post[512]
```

并带有：

```text
rsa_pub[512]
```

因此完整 round-trip 必须考虑的不只是：

```text
decrypt → modify → encrypt
```

还包括：

```text
digest
signature
length
header
footer
```

之间的关系。

---

## 7. Footer 也曾经是陷阱

项目早期曾把 OEM footer 理解为更短的尾部数据，后来修正为：

```text
13 bytes
```

这看起来是一个很小的差异，却是典型的 firmware reverse-engineering 问题：

```text
99% 的格式正确
≠
可验证 round-trip
```

一个尾部长度错误，就足以说明模型还没有闭合。

---

## 8. 解密以后终于看到标准 Linux firmware

解密 payload 大小：

```text
21,496,156 bytes
```

内部出现 FIT image。

FIT totalsize：

```text
0x27fe5f
```

其中包括：

```text
kernel@1
    compression: lzma
    load:  0x41080000
    entry: 0x41080000

fdt@1
    device tree

rootfs
    SquashFS
    offset around 0x2a0000
```

这一步以后，研究对象才从 D-Link proprietary container 转换成我们熟悉的：

```text
Linux kernel
DTB
SquashFS
root filesystem
kernel modules
OEM userspace
```

---

## 9. Device Tree 是硬件地图的重要来源

从 OEM DTB 可以继续恢复 flash partition 和 board resources。

项目观察到的 partition 包括：

```text
Preloader
ATF
Bootloader
Config
Factory
```

OpenWrt runtime 中 `/proc/mtd` 也提供了对应验证。

这让 OEM image analysis 与真实设备 flash layout 连接起来。

---

## 10. 为什么必须做 Round-Trip

“我可以解密”不是终点。

真正强的验证方式是：

```text
OEM image
    ↓
parse
    ↓
decrypt
    ↓
recover plaintext
    ↓
reconstruct container
    ↓
re-encrypt
    ↓
rebuild hashes/signatures/metadata
    ↓
compare / verify
```

只有 round-trip 成功，才说明我们理解的是**格式关系**，而不是碰巧提取出了 rootfs。

本项目最终验证了：

```text
hash relationships
signature relationships
container structure
```

并发现 OpenWrt host tool 中已经存在与 DIR-X3260 相关的 `dlink-sge-image` 支持和常量，可以与逆向结果交叉验证。

---

## 11. 不要让现有工具替代逆向证据

找到一个能够处理镜像的现成工具当然非常有价值。

但工程上应该区分两件事：

```text
工具能处理这个 image
```

与：

```text
我们知道它为什么能处理
```

DIR-X3260 项目没有因为发现 `dlink-sge-image` 就停止分析。

相反，我们利用 OEM binary、header、rootfs 文件和 round-trip 结果去验证工具行为。

这样才能在工具失败、版本变化或格式出现边界情况时知道问题在哪里。

---

## 12. SHRS 逆向为什么影响后面的 Wi-Fi 研究

一旦 OEM rootfs 可以可靠提取，就能进入：

```text
/lib/modules/...
```

并分析 OEM MediaTek driver。

后来的 MT7622 和 MT7915 研究都依赖这一步。

例如我们最终能够反汇编 OEM：

```text
mt7622_mt_wifi.ko
```

继续追踪：

```text
DriverOwn
FirmwareOwn
WBSYS / IOC
MCU
Andes LED command
```

所以 SHRS 并不是项目外围的“固件解包工作”。

它实际上打开了后续整个 reverse-engineering chain。

---

## 13. 一个更可靠的固件格式逆向方法

推荐顺序：

```text
1. 记录原始 image 的 size / SHA256
2. 确认 magic / header boundary
3. 解析长度字段
4. 识别 entropy / encrypted region
5. 从 OEM userspace 找 decrypt/verify implementation
6. 恢复 AES 参数
7. 恢复 digest relationships
8. 恢复 signature relationships
9. 确认 footer
10. 解密 payload
11. 解析 FIT / DTB / SquashFS
12. 做 round-trip
13. 用 OEM tool / OpenWrt tool 交叉验证
```

不要一开始就急着“改一个字节再刷”。

先证明格式模型闭合。

---

## 14. 证据边界

本文有意区分：

### 本项目直接验证

```text
SHRS header layout
0x6dc payload boundary
AES-128-CBC
SHA-512 relationships
4096-bit RSA material/signatures
13-byte footer
decrypted payload structure
FIT / DTB / SquashFS
round-trip verification
OEM imgdecrypt/public.pem/enk.txt evidence
```

### 不应仅凭经验扩大声称

例如：

```text
所有 D-Link SHRS 产品都完全相同
所有硬件 revision 都使用同一套 key/signature policy
其他型号可以直接套用 DIR-X3260 参数
```

DIR-X3260 A1 上验证的事实，不应自动升级成整个 D-Link 产品线的通用结论。

---

## 结语

固件逆向最危险的状态不是“完全看不懂”。

而是：

> **已经能解包，所以误以为自己完全理解了格式。**

DIR-X3260 的 SHRS 研究最终要求每一层都能互相解释：

```text
header
→ encryption
→ digest
→ RSA
→ footer
→ FIT
→ DTB
→ SquashFS
→ OEM implementation
→ round-trip
```

当这些证据闭合以后，OEM firmware 才从一个黑盒文件真正变成了可以研究、验证和复现的工程对象。
