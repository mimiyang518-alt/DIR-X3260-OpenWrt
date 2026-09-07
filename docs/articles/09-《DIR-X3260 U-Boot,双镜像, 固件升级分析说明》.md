# DIR-X3260 U-Boot / 双镜像 / 固件升级分析说明

## 1. 目的

本文件承接：

`DIR-X3260-Firmware-Structure-Crypto-Key-Analysis.md`

目标是把：

```text
PC 官方固件
    ↓
SHRS / AES / SHA-512 / RSA
    ↓
解密 payload
    ↓
FIT / kernel / rootfs
    ↓
Flash MTD
    ↓
U-Boot
    ↓
main / recovery 双镜像
    ↓
Linux / Web Upgrade
```

统一起来。

特别注意：

**SHRS 是“固件文件格式/密码学层”；双镜像是“设备启动与 Flash 布局层”。不能因为官方文件可以解密，就直接推断 main/recovery 的具体 Flash 地址。地址必须以实际 MTD/U-Boot 数据为准。**

---

# 2. 已确认的 SHRS 层

OpenWrt `dlink-sge-image` 当前明确支持：

```text
DIR-X3260
```

并使用：

```text
AES-128-CBC
SHA-512
RSA-4096
```

工具定义：

```text
HEADER_LEN = 1756 = 0x6DC
```

官方格式：

```text
0x0000  SHRS
0x0004  payload_length_before
0x0008  payload_length_post
0x000C  salt              16 bytes
0x001C  md_vendor         64 bytes
0x005C  md_before         64 bytes
0x009C  md_post           64 bytes
0x00DC  rsa_pub           512 bytes
0x02DC  rsa_sign_before   512 bytes
0x04DC  rsa_sign_post     512 bytes
0x06DC  encrypted payload
```

OpenWrt 源码明确按照上述顺序读取 header，然后从 `0x6DC` 开始 AES 解密。`rsa_pub` 在当前实现中被跳过；生成镜像时该区域被零填充。citeturn0search0turn2view0

---

# 3. 一个重要修正：payload length

这里要严格区分两个长度：

```text
payload_length_before
```

表示：

```text
原始 payload 的实际长度
```

而：

```text
payload_length_post
```

表示：

```text
AES 加密后 payload 的长度
```

源码使用：

```text
pad_len = 16 - (payload_length_before % 16)
```

如果原始长度刚好是 16 的整数倍，仍然追加一个完整的 16-byte padding block。

因此：

```text
payload_length_post
=
payload_length_before + pad_len
```

这也是为什么不能简单：

```text
file_size - 1756
```

就当成原始 payload 长度。

另外，整个文件最后还有：

```text
00 00 00 00 30
```

5-byte footer。

所以：

```text
完整文件 =
1756-byte SHRS header
+
payload_length_post-byte encrypted payload
+
5-byte footer
```

这点由当前 OpenWrt 实现直接确认。citeturn2view0

---

# 4. DIR-X3260 vendor key

GPL 中：

```text
DIR-X3260_GPL_Release/
MTK7621_AX1800_BASE/
vendors/DIR-X3260/imgkey/enk.txt
```

对应：

```text
NF5yKy10JTl+bSkhNj1kTTIkI3FhIyUsJDU0czMyZmR6Jl4jMzI4KjA2Mg==
```

OpenWrt 的处理不是简单 Base64 decode。

实际：

```text
enk.txt
   ↓
Base64 decode
   ↓
8-byte block deinterleave
   ↓
取前 16 bytes
   ↓
vendor_key
```

OpenWrt 源码明确给出了 DIR-X3260 的 `enk.txt` 和 8×8 interleave pattern。citeturn0search6

---

# 5. AES 解密链

最终：

```text
vendor_key = 16 bytes
salt       = header[0x0C:0x1C]
```

AES：

```text
AES-128-CBC
Key = vendor_key
IV  = salt
```

源码明确执行：

```text
EVP_DecryptInit_ex(... vendor_key, aes_iv)
EVP_CIPHER_CTX_set_padding(..., 0)
```

也就是说：

```text
OpenSSL 默认 PKCS#7 padding
```

并没有被直接使用。

设备格式采用自己的：

```text
zero padding
```

方式。citeturn2view0

---

# 6. SHA-512 三条验证链

原始 payload：

```text
P
```

vendor key：

```text
K
```

加密 payload：

```text
C
```

则：

```text
md_before = SHA512(P)

md_vendor = SHA512(P || K)

md_post   = SHA512(C)
```

其中：

```text
||
```

表示字节拼接。

这是判断解密结果是否正确的重要依据。

---

# 7. RSA 签名

DIR-X3260 使用：

```text
RSA key length = 512 bytes
              = 4096 bits
```

签名：

```text
signature_before = RSA_sign(SHA512, md_before)

signature_post   = RSA_sign(SHA512, md_post)
```

当前 OpenWrt 工具在生成时：

```text
RSA private key
       ↓
签名 md_before
       ↓
512 bytes

RSA private key
       ↓
签名 md_post
       ↓
512 bytes
```

因此 SHRS 的两个 RSA 签名并不是“签整个固件文件”。

而是分别保护：

```text
原始 payload digest
加密 payload digest
```

OpenWrt 源码直接实现了这两个签名步骤。citeturn2view0

---

# 8. 这意味着什么？

因此我们现在可以把：

```text
“加密”
```

和：

```text
“防篡改/认证”
```

分开理解。

### AES

解决：

```text
别人不知道 vendor_key 时，
不能直接看到 payload。
```

### SHA-512

解决：

```text
检查数据是否发生改变。
```

### RSA

解决：

```text
验证 digest 是否由对应 private key 签名。
```

所以：

```text
AES ≠ RSA
加密 ≠ 签名
vendor_key ≠ RSA private key
key.pem password ≠ vendor_key
```

---

# 9. U-Boot 层

现在进入真正与“双镜像”有关的部分。

之前我们从 DIR-X3260 的 U-Boot binary 中已经找到：

```text
Starting dual image checking
```

以及：

```text
read image from main image
```

这类字符串。

这些字符串非常重要，因为它们说明：

```text
U-Boot 本身存在 dual-image 检查逻辑。
```

但是仅凭字符串还不能证明：

```text
main = MTD partition X
recovery = MTD partition Y
```

也不能仅凭字符串证明：

```text
Reset = 强制 recovery
```

这些必须进一步从反汇编和运行时实验确认。

---

# 10. U-Boot 双镜像应该拆成五个问题

### 问题 A：两个 image 在哪里？

寻找：

```text
main image offset
recovery image offset
```

来源可能是：

```text
U-Boot 常量
device tree
MTD partition
environment variable
NAND/Flash layout
```

---

### 问题 B：U-Boot 怎么判断 image 有效？

寻找：

```text
checksum
CRC
SHA
magic
uImage header
FIT
signature
version
size
```

特别要把：

```text
SHRS signature
```

和：

```text
U-Boot image validity
```

区分开。

---

### 问题 C：U-Boot 怎么选择 image？

典型逻辑可能类似：

```text
检查 main
   |
   +-- valid → boot main
   |
   +-- invalid
           |
           v
       检查 recovery
           |
           +-- valid → boot recovery
```

但目前对于 DIR-X3260：

**这只能作为待验证模型，不能当成最终结论。**

---

### 问题 D：Reset 键是否改变选择？

需要确认：

```text
normal boot
```

和：

```text
boot + Reset pressed
```

之间到底改变的是：

```text
image selection
```

还是：

```text
boot delay
```

或者：

```text
进入 recovery HTTP/TFTP
```

这三个完全不同。

---

### 问题 E：升级到底写哪个 image？

这是最关键的问题。

例如：

```text
当前 boot main
       |
       | firmware upgrade
       v
写 recovery
       |
       v
改变 active flag
       |
       v
下一次启动 recovery
```

或者：

```text
当前 boot main
       |
       v
直接覆盖 main
```

两者差别非常大。

---

# 11. 为什么你之前“刷旧固件后 admin 密码没有变化”很重要

这个现象不能直接证明：

```text
password 在 recovery 分区
```

反而更应该首先检查：

```text
MTD partition
config
nvram
factory
persistent data
overlay
```

因为官方 firmware upgrade 往往不会覆盖全部 Flash。

所以模型应该是：

```text
                    SPI-NAND / NAND
                         |
        +----------------+----------------+
        |                |                |
     bootloader       firmware A       firmware B
        |                |                |
        |                |                |
        +----------------+----------------+
                         |
                   persistent data
                         |
                 admin configuration
```

实际结构必须通过：

```bash
cat /proc/mtd
```

和：

```bash
dmesg | grep -i mtd
```

以及 U-Boot 分区定义共同确认。

---

# 12. 下一轮实验：先不要刷机

现在最安全的办法是先采集设备信息。

在当前 OpenWrt DIR-X3260 上执行：

```bash
echo '===== /proc/mtd ====='
cat /proc/mtd

echo
echo '===== cmdline ====='
cat /proc/cmdline

echo
echo '===== mtd devices ====='
ls -l /dev/mtd*

echo
echo '===== dmesg mtd ====='
dmesg | grep -Ei 'mtd|spi-nand|nand|ubi'

echo
echo '===== partitions ====='
cat /proc/partitions
```

然后：

```bash
echo '===== device tree partitions ====='
find /proc/device-tree -type f \
  \( -name label -o -name reg -o -name compatible \) \
  2>/dev/null | sort
```

---

# 13. 如果可以进入 U-Boot

这是下一阶段最重要的数据。

在 U-Boot：

```text
printenv
```

重点寻找：

```text
bootcmd
bootargs
bootdelay
bootcount
bootlimit
image
main
recovery
dual
upgrade
partition
mtd
kernel
rootfs
```

Windows/Ubuntu 端也可以保存：

```text
printenv
```

的完整输出。

不要只截取几行。

---

# 14. U-Boot binary 静态分析

我们之前已经有：

```text
u-boot.bin
uboot_arm32_vma41e.asm
```

下一步不要再盲目搜索字符串。

应该针对：

```text
Starting dual image checking
```

找它的交叉引用。

例如：

```bash
grep -n -B 20 -A 50 \
'Starting dual image checking' \
uboot_arm32_vma41e.asm
```

如果反汇编里存在对应地址：

```text
41xxxxxx <...>
```

就从该地址开始向前/向后追踪：

```text
函数入口
   ↓
字符串引用
   ↓
比较/判断
   ↓
Flash read
   ↓
image validity
   ↓
branch
```

最终要恢复成类似 C 代码：

```c
if (check_main_image() == VALID) {
        boot_main();
} else {
        boot_recovery();
}
```

或者：

```c
if (reset_pressed()) {
        boot_recovery();
} else {
        boot_main();
}
```

但在真正找到汇编证据之前，不能预设是哪一种。

---

# 15. 特别关注 ARM32 U-Boot

你之前已经得到：

```text
uboot_arm32_vma41e.asm
```

并且使用：

```text
41e00000
```

作为反汇编 VMA。

这非常有价值。

下一步重点不是重新 dump U-Boot，而是：

```text
找到 dual-image 函数
        ↓
确定函数地址
        ↓
确定调用者
        ↓
确定 Flash 地址参数
        ↓
确定 validity check
        ↓
确定 branch condition
```

这可以直接回答：

> DIR-X3260 到底如何选择 main/recovery。

---

# 16. 一个特别重要的区别：Factory firmware 与 Recovery image

不能把下面三个概念混为一谈：

```text
官方升级文件
```

```text
Flash 中的 recovery image
```

```text
U-Boot recovery mode
```

它们可能分别是：

```text
官方文件：
SHRS + encrypted payload

Flash：
kernel/rootfs image

Recovery mode：
U-Boot 提供的特殊启动/升级路径
```

因此即使官方 `.bin` 被称为：

```text
recovery
```

也不意味着：

```text
整个官方 .bin = Flash recovery partition
```

需要实际解析。

---

# 17. OpenWrt 侧的另一个线索

OpenWrt 当前 firmware-utils 的 CMake 构建中已经把：

```text
dlink-sge-image
```

作为独立 firmware utility 编译，并链接 OpenSSL crypto。citeturn0search1

OpenWrt 的 image build 也通过：

```text
Build/dlink-sge-image
```

把生成的 factory image 交给：

```text
dlink-sge-image
```

进行 SGE 包装。citeturn0search2

因此：

```text
OpenWrt factory image
       ↓
payload
       ↓
dlink-sge-image
       ↓
SHRS encrypted factory image
```

这进一步证明：

**SHRS 是 factory-image 包装层，而不是 Linux kernel 本身的启动格式。**

---

# 18. 当前证据等级

为了避免后面研究时把“推测”当“事实”，以后统一使用：

### A — 已从源码确认

例如：

```text
DIR-X3260 使用 dlink-sge-image
AES-128-CBC
SHA-512
RSA-4096
HEADER_LEN = 1756
vendor_key derivation
```

### B — 已从设备实测确认

例如：

```text
/proc/mtd
dmesg
MTD partition
实际 boot 行为
实际 Reset 行为
```

### C — 从 U-Boot 反汇编确认

例如：

```text
main offset
recovery offset
boot selection
validity check
```

### D — 推测

例如：

```text
Reset 一定启动 recovery
```

在没有实验/反汇编证据之前只能标记为：

```text
HYPOTHESIS
```

---

# 19. 目前最值得做的工作

按照优先级：

```text
① 保存完整 U-Boot binary
        ↓
② 完整反汇编
        ↓
③ 定位 dual image checking
        ↓
④ 找 main/recovery Flash offset
        ↓
⑤ 找 image validity 函数
        ↓
⑥ 找 Reset GPIO/按键判断
        ↓
⑦ 对照 /proc/mtd
        ↓
⑧ 对照实际 Flash dump
        ↓
⑨ 恢复双镜像完整结构
```

然后才研究：

```text
Web Upgrade
```

因为 Web Upgrade 最终一定要回答：

```text
它把哪个 payload
写到哪个 Flash 分区？
```

---

# 20. 最终目标

最终我们希望把 DIR-X3260 完整启动链恢复成这样：

```text
                 官方 firmware .bin
                         |
                         v
                  +-------------+
                  |    SHRS     |
                  +-------------+
                         |
              AES-128-CBC decrypt
                         |
                         v
                  firmware payload
                         |
                         v
                     FIT/UBI
                         |
                         v
                  kernel + rootfs
                         |
                         v
              +---------------------+
              | Flash partition map |
              +---------------------+
                  |             |
                  v             v
               MAIN          RECOVERY
                  \             /
                   \           /
                    +---------+
                         |
                       U-Boot
                         |
                 dual-image check
                         |
             +-----------+-----------+
             |                       |
          MAIN valid             MAIN invalid
             |                       |
             v                       v
          boot MAIN             boot RECOVERY
                                     |
                                     v
                                  Linux
```

但这里最后这一张图中：

```text
MAIN
RECOVERY
```

的具体地址、大小、选择条件和 Reset 行为，**现在仍然必须通过 U-Boot 反汇编 + `/proc/mtd` + 实机实验来最终确认。**

---

# 21. 下一步直接操作

把你之前的：

```text
u-boot.bin
uboot_arm32_vma41e.asm
```

以及现在设备执行：

```bash
cat /proc/mtd
cat /proc/cmdline
dmesg | grep -Ei 'mtd|nand|ubi'
```

的结果发给我。

我下一步可以直接从：

```text
Starting dual image checking
```

这个字符串开始，**逐条追 ARM 汇编交叉引用**，把 DIR-X3260 的双镜像选择函数反推出接近 C 代码的形式。

这一步比继续猜分区结构更关键。
