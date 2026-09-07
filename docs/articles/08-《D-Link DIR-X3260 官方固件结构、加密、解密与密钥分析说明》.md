# D-Link DIR-X3260 官方固件结构、加密/解密与密钥分析说明

> 项目对象：D-Link DIR-X3260 A1 / REVA  
> 文档用途：用于后续 OpenWrt 移植、官方固件逆向分析、固件解包/重封装、升级流程研究，以及双镜像启动机制分析。  
> 状态：截至 2026-09-07  
>
> **重要区分：**“固件文件加密”和“固件签名”是两件不同的事情。DIR-X3260 的 SGE/D-Link 镜像机制同时涉及 AES-128-CBC、SHA-512 摘要和 RSA-4096 签名。

---

## 1. 官方固件资料

D-Link 官方历史固件目录目前可以看到：

- `DIR-X3260_REVA-FIRMWARE_v1.00.zip`
- `DIR-X3260_REVA-FIRMWARE_v1.01B05.zip`
- `DIR-X3260_REVA-FIRMWARE_v1.02B02.zip`
- `DIRX3260_REVA_FIRMWARE_104B01.zip`

官方支持页面已经标记 DIR-X3260 为 EOL，支持结束日期为 2024-03-31。

建议本项目至少保留以下原始文件：

```text
DIR-X3260/
├── official/
│   ├── DIR-X3260_REVA-FIRMWARE_v1.00.zip
│   ├── DIR-X3260_REVA-FIRMWARE_v1.01B05.zip
│   ├── DIR-X3260_REVA-FIRMWARE_v1.02B02.zip
│   └── DIRX3260_REVA_FIRMWARE_104B01.zip
├── extracted/
├── decrypted/
├── rebuilt/
├── uboot/
├── gpl/
└── analysis/
```

---

# 2. 目前确认的官方镜像外层结构

DIR-X3260 使用的 D-Link SGE 镜像格式可以由 OpenWrt `firmware-utils` 中的 `dlink-sge-image` 复现。

镜像最前面有：

```text
SHRS
```

因此首先可以用：

```bash
xxd -l 64 firmware.bin
```

或者：

```bash
strings -a -t x firmware.bin | head
```

检查是否存在 `SHRS`。

---

# 3. SHRS Header

目前 OpenWrt 的实现明确给出了：

```text
HEADER_LEN = 1756 bytes
```

结构如下：

| Offset | Size | 内容 |
|---:|---:|---|
| `0x0000` | 4 | Magic = `SHRS` |
| `0x0004` | 4 | 解密前 payload 长度 |
| `0x0008` | 4 | 加密/填充后 payload 长度 |
| `0x000C` | 16 | Salt |
| `0x001C` | 64 | SHA-512 vendor digest |
| `0x005C` | 64 | SHA-512 plaintext digest |
| `0x009C` | 64 | SHA-512 ciphertext digest |
| `0x00DC` | 512 | RSA public-key 保留区域 |
| `0x02DC` | 512 | RSA signature of plaintext digest |
| `0x04DC` | 512 | RSA signature of ciphertext digest |
| `0x06DC` | ... | AES-128-CBC 加密 payload |
| payload 后 | 5 | footer：`00 00 00 00 30` |

总 Header：

```text
0x06DC = 1756
```

因此：

```text
SHRS header = 1756 bytes
encrypted payload starts at 0x06DC
```

---

# 4. 加密算法

DIR-X3260 不是简单的“整个 ZIP 用 AES 加密”。

真正的结构是：

```text
原始 firmware payload
        |
        | SHA-512
        v
 plaintext digest
        |
        +-----------------------------+
        |                             |
        | AES-128-CBC                 | SHA-512(payload + vendor_key)
        |                             |
        v                             v
encrypted payload              vendor digest
        |
        | SHA-512
        v
ciphertext digest
```

AES 使用：

```text
AES-128-CBC
```

并且：

```text
IV = salt
Key = 16-byte vendor_key
```

代码中明确使用：

```c
EVP_EncryptInit_ex(
    aes_ctx,
    aes128,
    NULL,
    &vendor_key[0],
    aes_iv
);
```

并且关闭 OpenSSL 默认 padding：

```c
EVP_CIPHER_CTX_set_padding(aes_ctx, 0);
```

因此固件工具自己进行块对齐。

---

# 5. DIR-X3260 vendor_key 的来源

这是本项目最关键的部分之一。

DIR-X3260 不是使用旧 D-Link 型号的 legacy vendor-key 算法。

OpenWrt 源码明确把 DIR-X3260 单独归入：

```text
generate_vendorkey_dimgkey()
```

处理流程：

```text
DIR-X3260 enk.txt
       |
       | Base64 decode
       v
binary data
       |
       | deinterleave
       v
前 16 bytes
       |
       v
vendor_key
```

---

# 6. DIR-X3260 enk.txt

OpenWrt `dlink-sge-image.h` 中公开了 DIR-X3260 对应的 `enk.txt` 内容：

```text
NF5yKy10JTl+bSkhNj1kTTIkI3FhIyUsJDU0czMyZmR6Jl4jMzI4KjA2Mg==
```

来源注释明确指出：

```text
DIR-X3260_GPL_Release/
MTK7621_AX1800_BASE/
vendors/
DIR-X3260/
imgkey/
enk.txt
```

也就是说，这个值并不是通过猜测得到的，而是来自 D-Link GPL 发布包中的 `imgkey/enk.txt`。

---

# 7. deinterleave 算法

Base64 解码后，并不是直接取前 16 bytes。

还要经过 8-byte block 的 interleave/deinterleave。

源码定义：

```text
INTERLEAVE_BLOCK_SIZE = 8
```

pattern：

```text
{2, 5, 7, 4, 0, 6, 1, 3}
{7, 3, 2, 6, 4, 5, 1, 0}
{5, 1, 6, 7, 3, 0, 4, 2}
{0, 3, 7, 6, 5, 4, 2, 1}
{1, 5, 7, 0, 3, 2, 6, 4}
{3, 6, 2, 5, 4, 7, 1, 0}
{6, 0, 5, 1, 3, 4, 2, 7}
{4, 6, 7, 3, 2, 0, 1, 5}
```

算法每 8 bytes 使用一行 pattern。

因此：

```text
Base64 decode
    ↓
deinterleave
    ↓
16-byte vendor_key
```

这也是为什么直接对 `enk.txt` 做 MD5/SHA 或直接 Base64 decode 后取 16 bytes 会得到错误结果。

---

# 8. RSA 密钥

DIR-X3260 同时使用 RSA 签名。

源码定义：

```text
RSA_KEY_LENGTH_BYTES = 512
```

即：

```text
512 bytes = 4096 bits
```

因此这里是 RSA-4096。

Header 中存在两个实际签名：

```text
RSA signature #1
    SHA-512(plaintext)

RSA signature #2
    SHA-512(ciphertext)
```

另外还有一个 512-byte 区域：

```text
rsa_pub
```

在当前 OpenWrt 工具实现中被清零/跳过，因此不能简单把它理解成“header 中携带完整 RSA public key”。

---

# 9. DIR-X3260 RSA private key

OpenWrt `dlink-sge-image.h` 中包含 DIR-X3260 对应的 RSA private key。

来源注释：

```text
DIR-X3260_GPL_Release/
MTK7621_AX1800_BASE/
vendors/DIR-X3260/
imgkey/key.pem
```

该 PEM 本身使用 AES-256-CBC 加密。

头部形式：

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-256-CBC,...
```

OpenWrt 工具中用于读取该 PEM 的 passphrase 是：

```text
12345678
```

注意：

```text
这是 key.pem 文件本身的保护密码
不是 AES firmware vendor_key
也不是路由器管理员密码
也不是 Wi-Fi 密码
```

这几个概念必须严格区分。

---

# 10. 三种“密钥”必须分开

DIR-X3260 固件分析中最容易混淆的是下面三个东西：

## A. vendor_key

用途：

```text
AES-128-CBC 固件 payload 加密/解密
```

来源：

```text
enk.txt
    ↓ Base64
    ↓ deinterleave
vendor_key
```

长度：

```text
16 bytes
```

---

## B. RSA private key

用途：

```text
生成 firmware signature
```

用于：

```text
SHA512(plaintext)
SHA512(ciphertext)
```

然后使用 RSA private key 签名。

---

## C. key.pem passphrase

用途：

```text
解锁 GPL 中的加密 RSA private key PEM
```

当前 OpenWrt 实现：

```text
12345678
```

---

# 11. SHA-512 验证链

固件生成时计算三个 digest：

```text
md_before
    = SHA512(original_payload)

md_post
    = SHA512(encrypted_payload)

md_vendor
    = SHA512(original_payload + vendor_key)
```

因此可以建立如下完整验证关系：

```text
                  +--------------------+
                  | original payload   |
                  +---------+----------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
          SHA512        AES-128-CBC    append key
              |             |             |
              v             v             v
        md_before    encrypted payload  payload+key
                            |             |
                            v             v
                        md_post       md_vendor
```

---

# 12. RSA 签名关系

RSA 签名不是直接对整个 firmware 文件进行。

当前工具的实现是：

```text
RSA_sign(
    SHA512(original_payload)
)
```

以及：

```text
RSA_sign(
    SHA512(encrypted_payload)
)
```

因此 header 中存在：

```text
signature_before
signature_post
```

这也是判断固件是否被修改的重要依据。

---

# 13. 解密流程

标准解密流程：

```text
official encrypted image
        |
        v
检查 SHRS
        |
        v
读取 payload_length_before
payload_length_post
        |
        v
读取 salt
        |
        v
生成 DIR-X3260 vendor_key
        |
        v
AES-128-CBC decrypt
        |
        v
去掉 padding
        |
        v
得到原始 payload
        |
        +--> SHA512 校验
        |
        +--> RSA signature 校验
```

OpenWrt 工具的命令形式：

```bash
dlink-sge-image DIR-X3260 input.bin output.bin -d
```

其中：

```text
input.bin  = 官方加密镜像
output.bin = 解密后的 firmware payload
```

---

# 14. 编译 dlink-sge-image

在 OpenWrt 源码树中：

```bash
cd ~/src/openwrt
```

可以查找：

```bash
find . -name 'dlink-sge-image*'
```

通常源码位于：

```text
tools/firmware-utils/src/
```

或者构建后的 host 工具目录。

当前我们之前使用过的 OpenWrt build tree 中也存在：

```text
build_dir/host/u-boot-2026.04/tools/
```

如果工具已经编译：

```bash
find ~/src/openwrt -name dlink-sge-image -type f
```

---

# 15. 官方镜像分析推荐流程

不要直接修改原始官方文件。

建议：

```text
official/
    ↓
sha256
    ↓
copy
    ↓
identify
    ↓
decrypt
    ↓
binwalk
    ↓
dumpimage / unsquashfs
```

例如：

```bash
sha256sum DIR-X3260_REVA-FIRMWARE_v1.02B02.bin
file DIR-X3260_REVA-FIRMWARE_v1.02B02.bin
xxd -l 64 DIR-X3260_REVA-FIRMWARE_v1.02B02.bin
```

如果文件以：

```text
SHRS
```

开头，则进入 SGE 加密镜像分析流程。

---

# 16. 解密后再分析 FIT / kernel / rootfs

我们之前对 DIR-X3260 固件已经观察到：

```text
ARM64
Linux
FIT image
```

并且 OpenWrt 目标为：

```text
medIATEK / mt7622
aarch64_cortex-a53
```

解密之后建议继续：

```bash
file decrypted.bin
binwalk decrypted.bin
strings -a decrypted.bin | head -100
```

如果出现 FIT：

```bash
dumpimage -l decrypted.bin
```

然后：

```bash
dumpimage -T flat_dt -i decrypted.bin -p <index> extracted.itb
```

进一步提取：

```bash
binwalk -e extracted.itb
```

rootfs 如果是 SquashFS：

```bash
unsquashfs -d squashfs-root filesystem.squashfs
```

---

# 17. 我们目前已经确认的 DIR-X3260 内部结构

根据之前的实际分析，已经观察到：

```text
D-Link official image
        |
        v
SGE / SHRS encrypted wrapper
        |
        | AES-128-CBC
        v
firmware payload
        |
        v
FIT / kernel / filesystem
        |
        +-- Linux kernel
        |
        +-- SquashFS
        |
        +-- /etc
        +-- /etc/config
        +-- /etc/init.d
        +-- /lib/modules
```

之前实际提取到的 rootfs 中还出现：

```text
mt76.ko
mt76-connac-lib.ko
mt7615-common.ko
mt7615e.ko
mac80211.ko
cfg80211.ko
```

以及：

```text
ahci_mtk.ko
bluetooth.ko
btmtk.ko
```

这些属于解密后的 Linux 文件系统层，不属于 SHRS 加密层。

---

# 18. 双镜像问题与 SHRS 的关系

必须特别注意：

```text
SHRS firmware encryption
```

和：

```text
DIR-X3260 bootloader dual-image
```

不是同一个层次。

可以理解成：

```text
                D-Link firmware file
                       |
                 SHRS wrapper
                       |
              encrypted payload
                       |
                 bootloader
                       |
          +------------+------------+
          |                         |
       main image              recovery/backup
          |                         |
          +------------+------------+
                       |
                    Linux
```

因此后续分析必须分别研究：

### 层 1：PC 下载的官方固件

研究：

```text
SHRS
AES
SHA512
RSA
```

### 层 2：Bootloader

研究：

```text
U-Boot
双镜像选择
bootcmd
reset 键
image validity
active slot
```

### 层 3：Flash MTD

研究：

```text
/proc/mtd
partition layout
kernel
rootfs
config
nvram / factory
```

### 层 4：Linux upgrade

研究：

```text
/lib/upgrade/
/sbin/sysupgrade
web upgrade
HNAP
Management_Upgrade.js
```

这四层不能混在一起。

---

# 19. 关于“破解官方密钥”的准确结论

目前不应该表述为：

> “我们破解了 D-Link 的 RSA。”

更准确的说法是：

```text
DIR-X3260 的固件加密/签名格式已经被公开逆向并实现，
所需的 enk.txt、vendor-key derivation 方法以及对应 RSA
private key 已存在于 OpenWrt firmware-utils 的公开代码中。
```

OpenWrt 的 `dlink-sge-image` 已明确支持：

```text
DIR-X3260
```

并提供：

```text
decrypt
encrypt
signature verification
```

所以对于 DIR-X3260 来说，官方 SGE 镜像的加密层已经不是未知黑盒。

---

# 20. 当前研究的真正剩余问题

因此，我们后面的重点已经不应该继续放在“能不能解密官方 firmware”。

真正值得继续研究的是：

### ① 官方 firmware payload 的精确结构

确认：

```text
kernel
rootfs
device metadata
version
board ID
upgrade metadata
```

的准确偏移和关系。

### ② Web Upgrade 验证流程

重点：

```text
Management_Upgrade.js
SOAPFirmware_multi.js
HNAP
GetFirmwareValidation
FirmwareUpload
```

确认浏览器上传的到底是：

```text
SHRS encrypted image
```

还是：

```text
某种外层包装 + SHRS
```

### ③ Bootloader 对镜像的验证

重点研究：

```text
u-boot
image validation
RSA
SHA
SHRS
dual image
recovery
```

### ④ 双分区 / 双镜像选择机制

重点验证：

```text
main
recovery
active image
boot flag
reset key
upgrade target
```

### ⑤ 哪些数据不会被 firmware upgrade 覆盖

这与之前观察到：

```text
刷入旧 firmware 后 admin password 没有改变
```

的问题直接相关。

需要进一步对：

```text
MTD partition
U-Boot environment
config
factory
nvram
persistent overlay
```

做对应关系。

---

# 21. 推荐的项目目录

最终建议把 DIR-X3260 项目整理成：

```text
DIR-X3260-Analysis/
│
├── 00-original/
│   ├── v1.00/
│   ├── v1.01B05/
│   ├── v1.02B02/
│   └── v1.04B01/
│
├── 01-sge/
│   ├── dlink-sge-image.c
│   ├── dlink-sge-image.h
│   ├── enk.txt
│   ├── vendor-key/
│   └── rsa/
│
├── 02-decrypted/
│
├── 03-fit/
│
├── 04-kernel/
│
├── 05-rootfs/
│
├── 06-uboot/
│
├── 07-mtd/
│
├── 08-dual-image/
│
├── 09-web-upgrade/
│
├── 10-openwrt/
│
└── DOCUMENTATION/
    ├── DIR-X3260-Firmware-Structure.md
    ├── DIR-X3260-SGE-Crypto.md
    ├── DIR-X3260-Key-Analysis.md
    ├── DIR-X3260-Uboot-DualImage.md
    └── DIR-X3260-Web-Upgrade.md
```

---

# 22. 参考资料

1. D-Link 官方 DIR-X3260 REVA firmware archive。
2. OpenWrt `firmware-utils/src/dlink-sge-image.c`
3. OpenWrt `firmware-utils/src/dlink-sge-image.h`
4. OpenWrt-devel 关于 `dlink-sge-image` 的提交说明。
5. DIR-X3260 GPL release 中的 `vendors/DIR-X3260/imgkey/enk.txt` 与 `key.pem`。

---

## 23. 最终结论

目前 DIR-X3260 官方固件加密链可以归纳为：

```text
D-Link GPL imgkey/enk.txt
            |
            | Base64 decode
            v
      deinterleave
            |
            v
    16-byte vendor_key
            |
            v
     AES-128-CBC
       IV = salt
            |
            v
    encrypted firmware
            |
            +--> SHA512(ciphertext)
            |
            +--> RSA-4096 signature
```

原始 payload 同时：

```text
SHA512(payload)
```

并计算：

```text
SHA512(payload + vendor_key)
```

最终 SHRS header 为：

```text
1756 bytes = 0x6DC
```

所以：

```text
0x0000  SHRS
0x0004  payload length before
0x0008  payload length after
0x000C  salt
0x001C  SHA512 vendor
0x005C  SHA512 plaintext
0x009C  SHA512 ciphertext
0x00DC  RSA public/reserved area
0x02DC  RSA signature plaintext
0x04DC  RSA signature ciphertext
0x06DC  encrypted payload
```

这套结构已经足够作为我们后续分析 DIR-X3260 官方固件、U-Boot、双镜像和 OpenWrt 刷机流程的统一密码学基础。
