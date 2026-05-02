
# Pico FIDO2 项目指南

本项目致力于将您的 **Raspberry Pi RP235x** 或 **ESP32** 微控制器，锻造成集 **FIDO Passkey** 与 **OpenPGP 智能卡** 于一体的安全利器。它既能作为标准的 USB 通行密钥（Passkey）进行身份验证，亦可作为智能卡处理各类加密任务。

本项目源自 **pico-fido2** 固件于 2025 年 12 月 9 日发布的最后一个“社区版”分支。在该原仓库被删除并转为商业模式之前，我们及时保留了这一火种。

---

## 受支持的硬件平台

| 特性 | RP2040 | RP2350 | ESP32-S2 | ESP32-S3 |
|---|---|---|---|---|
| 处理器 | 双核 Cortex-M0+ | **双核 Cortex-M33** | 单核 Xtensa | 双核 Xtensa |
| 核心绑定 | 支持 | 支持 | 不支持 | 支持 |
| 操作系统 | 无 (Pico SDK) | 无 (Pico SDK) | FreeRTOS | FreeRTOS |
| 硬件标识 (ID) | `0` | `1` | `2` | `2` |

### 安全特性对比

目前，最完备的安全特性仅在 **RP2350** 平台上得到全面实现。

| 特性 | RP2350 | RP2040 | ESP32-S2 | ESP32-S3 |
|---|---|---|---|---|
| **安全启动 (Secure Boot)** | **全功能支持** (哈希校验、防调试、故障检测) | 无硬件支持 | 开发中 (`// TODO`) | 开发中 (`// TODO`) |
| **安全锁死 (Secure Lock)** | **支持** (密钥失效、页面锁定) | 不支持 | 不支持 | 不支持 |
| **OTP 硬件主密钥 (MKEK)** | **支持** (ECC 纠错、页面锁定) | 不支持 (Flash 明文) | 支持 (eFuse 写入锁) | 支持 (eFuse 写入锁) |
| **设备私钥 (OTP/eFuse)** | **支持** (含混淆与迁移保护) | 不支持 | 支持 | 支持 |
| **固件签名与回滚保护** | **支持** | 不支持 | 不支持 | 不支持 |
| **硬件加密加速** | SHA-256 | 无 | 多协议加速 | 多协议加速 |

---

## 功能特性

Pico FIDO2 涵盖以下核心功能：

### FIDO2 / U2F / WebAuthn (通行密钥)
*   **全协议支持：** 兼容 CTAP 2.1、WebAuthn 及 U2F 标准。
*   **安全扩展：** 支持 HMAC-Secret（用于离线加密）、CredProtect 等进阶扩展。
*   **物理交互：** 通过物理按键强制进行“用户存在确认”。
*   **验证机制：** 支持 PIN 码验证、驻留密钥（可发现凭据）管理。
*   **算法全覆盖：** 支持从 NIST P-256 到 Ed25519 等多种主流加密曲线。
*   **备份与恢复：** 支持 **24 位助记词** 备份；RP2350 独享 Flash 倾倒防护。
*   **大容量数据：** 支持 CredBlobs 及高达 2048 字节的 Large Blobs 存储。

### OATH / 动态令牌 (基于 YKOATH 协议)
*   **多模式：** 支持 **TOTP** (时间同步) 与 **HOTP** (计数同步)。
*   **模拟键盘 (HID)：** 模拟键盘输入，按下按键即可自动“打出”验证码。
*   **兼容性：** 完美适配 Yubico **YKMAN**、Nitrokey 等官方管理工具。

### OpenPGP 智能卡
*   **规范标准：** 遵循 OpenPGP Card v3.4 规范。
*   **三大槽位：** 独立管理 **签名 (Signature)**、**加密 (Encryption)** 与 **认证 (Authentication)** 密钥。
*   **高强度加密：** 支持高达 **4096 位 RSA** 以及现代 ECC 曲线。
*   **全流程受控：** 支持片上密钥生成、导入/导出，以及 Admin PIN 保护。
*   **生态互通：** 完美联动 GnuPG、SSH、S/MIME 等主流加密工具。

---

## 安全考量

**RP2350** 与 **ESP32-S3** 微控制器在开启“安全启动”后，可构建稳固的受信环境。通过将**主密钥 (MKEK)** 存储在**一次性可编程 (OTP)** 存储区中，任何外部非安全代码均无法读取。设备上所有的私钥均由此主密钥加密，即使 Flash 固件被物理提取（Dump），敏感数据依然固若金汤。

**郑重提醒：** **RP2040** 缺乏上述硬件级安全特性。存储在其 Flash 中的数据（包括私钥）极易被恶意读取。若基于 RP2040 的设备丢失，您的私钥存在泄露风险。

---

## 编译与烧录 (以 Raspberry Pi Pico 为例)

编译前请确保已安装交叉编译工具链及 Pico SDK。

```bash
# 克隆仓库及子模块
git clone https://github.com/youruser/pico-fido2
git submodule update --init --recursive
cd pico-fido2 && mkdir build && cd build

# 执行 CMake (可根据需要指定 VID/PID)
PICO_SDK_PATH=/你的路径/pico-sdk cmake .. -DPICO_BOARD=pico
make
```

### 载入固件：
1.  按住板载 **BOOTSEL** 键的同时插入 USB。
2.  将生成的 `pico_fido2.uf2` 拖入弹出的 U 盘。
3.  设备将自动重置，LED 闪烁即表示“甲胄已就绪”。
4.  您可以使用 [**picoforge**](https://github.com/librekeys/picoforge) 桌面应用进行图形化配置。

---

## 驱动与兼容性

Pico FIDO2 使用通用的 **HID** (用于 FIDO) 和 **CCID** (用于 OpenPGP) 驱动。这两类驱动预装于所有主流操作系统。无需额外驱动，即可像使用商业级 YubiKey 一样被浏览器和加密软件识别。

## 开源协议

本项目基于 **GNU Affero General Public License v3 (AGPLv3)** 协议发布。

> **致谢：** 本项目借鉴并整合了多个开源项目的精髓，详细列表请参阅 `LICENSE` 文件。