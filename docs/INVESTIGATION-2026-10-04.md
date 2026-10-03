# Secure Boot 最新调查报告

日期：2026-10-04（Asia/Shanghai）。证据来源：本机只读检查、用户在管理员 PowerShell 7.6.6 中回传的原始输出，以及此前 GitHub BIOS 排查记录。

## 当前结论

根因尚未确定。历史现象是：关闭 Secure Boot enforcement 后 Windows 能启动，开启后固件报告 `Windows Boot Manager boot failed.`，随后提示 `Default Boot Device Missing or Boot Failed.`。

本轮已确认固件 db 检出了 Windows UEFI CA 2023，实际 EFI 系统分区的 Windows Boot Manager 已由该 CA 签发，Windows Authenticode 检查为 `Valid`。因此，单纯“缺少 2023 证书”不符合当前证据；仅追加该证书的恢复 U 盘方案应降低优先级。

这些结果仍不能证明固件验证一定通过，也没有排除其他撤销规则、启动链后续组件或固件配置/兼容性问题。尚无依据断言 BIOS 损坏，或要求重装系统。

## 设备信息（本机读取）

| 项目 | 结果 |
|---|---|
| 整机/主板 | COLORFUL P16 Pro |
| CPU | Intel Core i9-13900HX |
| 内存 | 标称 16 GB；Windows 总量约 15.78 GiB |
| BIOS | SMBIOS 原文 `1.07.05COLO3`，日期 2025-08-14 |
| Windows | Windows 11 家庭中文版，25H2，`26200.9457` |
| 系统盘 | Disk 0，NVMe，GPT |
| EFI 系统分区 | Disk 0 Partition 2，`\Device\HarddiskVolume2` |

历史 BIOS 手抄记录写作 `1.07.05COL03`；与 SMBIOS 原文有 O/0 转录差异。选择固件更新前应核实实际型号、子型号和完整版本，不能据此猜测适用更新包。

## 当前固件与 Windows 状态

本机桌面 agent 的进程不是管理员，直接读取 UEFI 变量时返回访问被拒绝；下面的固件 db/dbx 检查结果来自用户管理员终端，而非读取失败后的默认值。

| 检查 | 结果 | 解释边界 |
|---|---|---|
| db 搜索 `Windows UEFI CA 2023` | `True` | 检出了该名称；尚未结构化解析完整数据库 |
| db 搜索 `Microsoft Windows Production PCA 2011` | `True` | 检出了旧 CA 名称 |
| dbx 搜索该 2011 CA 名称 | `False` | 未检出名称，不排除按证书/映像哈希撤销 |
| 注册表 `State\UEFISecureBootEnabled` | `0` | Windows 记录 Secure Boot 当前关闭 |
| 注册表 `Servicing\UEFICA2023Status` | `Updated` | 服务状态记录，不能替代当前固件验证 |
| 注册表 `AvailableUpdates` | `0` | 当前读取值，不能单独证明不存在固件故障 |
| `UEFICA2023Error` / `UEFICA2023ErrorEvent` | 未读取到值 | 不等于证明历史从未发生错误 |

## 实际启动文件与路径（用户管理员输出）

`bcdedit /enum {bootmgr}`：

```text
device      partition=\Device\HarddiskVolume2
path        \EFI\MICROSOFT\BOOT\BOOTMGFW.EFI
description Windows Boot Manager
```

实际文件路径：

```text
\\?\Volume{46cefd3f-da02-4a94-90ad-e10f228ef874}\EFI\Microsoft\Boot\bootmgfw.efi
```

`Get-AuthenticodeSignature -LiteralPath ...`：

```text
Status        : Valid
SignatureType : Authenticode
Issuer        : CN=Windows UEFI CA 2023, O=Microsoft Corporation, C=US
```

固件启动顺序为 Windows Boot Manager、EFI USB Device、EFI DVD/CDROM、EFI Network。当前未观察到 USB 或网络启动排在 Windows 之前。

本机另读到 `C:\Windows\Boot\EFI\bootmgfw.efi` 的签发者为 Microsoft Windows Production PCA 2011，版本为 `10.0.28000.342`。它是 Windows 目录副本，不能代替上述实际 EFI 分区文件；仅凭此差异不能诊断系统混装或确定故障。

## 其他线索

- 非管理员会话的 `Get-ScheduledTask` 未找到 `Secure-Boot-Update`；精确路径 `\Microsoft\Windows\PI\Secure-Boot-Update` 的 `schtasks /Query` 返回找不到路径。需在管理员 CLI 中复查。
- System 日志中存在多条 TPM-WMI 事件 1796，本轮最近取到的日期为 2026-08-15。XML 给出 `HResult=-2147024809`（`0x80070057`）；本地化消息没有明确更新类型。这是历史更新失败线索，尚未与当前启动失败建立因果关系。
- 本机 `C:\Windows\Boot\EFI\SecureBootRecovery.efi` 存在，Windows 签名检查为 `Valid`。本轮未运行该工具。
- 七彩虹官方页面搜索出现“windows证书更新说明”，但访问超时或 502；尚未核实针对本机的 BIOS 更新及适用性。

## 下一步

1. 在管理员 CLI 下确认执行进程确有管理员 token，然后只读检查 `Confirm-SecureBootUEFI`、`SecureBoot`、`SetupMode`、`PK`、`KEK`。用户上次仅回传固件启动顺序，没有回传这些变量结果；当前关闭 Secure Boot 时 `Confirm-SecureBootUEFI=False` 是预期。
2. 复查 Secure-Boot-Update 任务及服务状态，按日期关联事件，而非直接修复任务。
3. 根据结果再检查完整 db/dbx 结构、相关证书与映像哈希、后续 EFI 启动链及 OEM 固件兼容性。
4. 任何新的 BIOS 开关/重启测试前，确认历史现象是否仍可复现、期间是否改过设置，并准备设备加密恢复条件；本报告不授权这些修改。

本轮实际操作为读取和文档整理，没有修改 TPM、固件密钥、注册表、计划任务、EFI 启动文件或 BIOS。

## 官方参考

- [Get-SecureBootUEFI](https://learn.microsoft.com/en-us/powershell/module/secureboot/get-securebootuefi)：变量读取范围与管理员权限要求。
- [Secure Boot requirements and databases](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-secure-boot)：db/dbx 可存储证书或映像哈希，撤销规则优先。
- [Secure Boot update events](https://support.microsoft.com/en-us/servicing/os/windows/2022/06/secure-boot-db-and-dbx-variable-update-events)：更新与错误事件的解释。
- [Secure Boot troubleshooting guide](https://support.microsoft.com/en-us/servicing/os/secure-boot/2026/03/secure-boot-troubleshooting-guide)：服务状态、任务、固件限制和恢复工具的适用条件。
