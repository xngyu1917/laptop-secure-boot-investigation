# Secure Boot 最新调查报告

日期：2026-10-04（Asia/Shanghai）。证据来源：本机只读检查、用户在管理员 PowerShell 7.6.6 中回传的原始输出，以及此前 GitHub BIOS 排查记录。

## 当前结论（10:05 管理员阶段更新）

根因尚未确定。历史现象是：关闭 Secure Boot enforcement 后 Windows 能启动，开启后固件报告 `Windows Boot Manager boot failed.`，随后提示 `Default Boot Device Missing or Boot Failed.`。

第二阶段已在本机管理员进程执行 `TEMP-TASK.md` 的八项检查与条件判断。PK、KEK、db、dbx 均读取并完整解析成功；db 含 Windows UEFI CA 2023，实际 Boot Manager 的密码学签名、PE/COFF 摘要及到该 CA 的证书签名链验证通过，未命中当前 dbx 的 416 条 SHA-256 条目。固件 BootCurrent/Boot0001、物理 ESP 和 BCD 路径相互一致。Secure-Boot-Update 任务存在、已启用，今天最近运行结果为 0。

ESP 已在仓库之外完成 100 MiB 原始镜像与全部 149 个文件的备份，校验全部通过。当前没有证据支持 ESP/BCD 缺失、目标错误或上述启动映像的哈希撤销，因此按任务条件不生成 BCDBoot 修复脚本。具体结果、原始数据位置和检查限制见本报告下方“第二阶段”。

这些结果仍不能证明固件实际放行，也没有确认历史开启 Secure Boot 后的故障目前仍可复现。固件配置/实现、未检查组件及策略仍需进一步取证；尚无依据断言 BIOS 损坏或要求重装系统。仅追加 2023 证书的恢复方案继续降低优先级。

下方先保留第一阶段的来源与历史输出；“未回传”“需管理员复查”等当时状态由第二阶段结果更新，不能再当成当前未完成事项。

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

## 第一阶段固件与 Windows 状态

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

## 第一阶段提出的下一步（管理员读取已在第二阶段补齐）

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

## 第二阶段：执行 TEMP-TASK.md 的八项任务

执行日期：2026-10-04，主要取证时间 09:53–10:05（Asia/Shanghai）。本阶段事实均由本机工具亲自读取；上一阶段的用户回传与历史 BIOS 观察仍单独保留。

先检查执行 token：`IsAdministrator=True`；PowerShell `7.6.6`。固件变量读取成功，不再沿用第一阶段的访问拒绝结果。原始数据、采集/解析代码和备份只存本地仓库之外：

```text
C:\Users\shijie\AppData\Local\SecureBootInvestigation\20261004-095323-bdd7379f
```

该目录包含未经脱敏的序列号、UUID、固件数据和 BCD，未加入仓库、未上传。入口说明为 `READ-ME-FIRST.txt`。

### 八项任务完成情况

| 任务 | 本轮结果 | 检查边界 |
|---|---|---|
| 1. 硬件身份 | 已读取三类 SMBIOS 身份及系统信息，发现序列字段中的 `V360…` 线索 | 尚无完整、经 OEM 核实的 Clevo 平台 ID |
| 2. 全盘/ESP 盘点 | 1 块物理磁盘、5 个分区、1 个 ESP；固件原始启动项与设备映射核对完成 | 仅当前在线可见磁盘 |
| 3. 实际启动文件 | 检查 13 个 EFI 路径，记录签名、大小、时间、版本、整文件及 PE 哈希 | 未遍历全部启动驱动/代码完整性策略；WinRE 未实启动 |
| 4. 固件变量 | 6 个变量读取成功；4 个密钥数据库全部解析，无遗留字节；信任链及 dbx 比对完成 | Windows 侧静态检查，不模拟 OEM 固件实现 |
| 5. 更新历史 | 管理员复查任务、注册表、日志及 Boot Manager SVN 完成 | 日志只代表现存保留范围，不能证明完整更新历史 |
| 6. 恢复环境 | WinRE、恢复 BCD、WIM 元数据、实际恢复加载器及 OEM 配置已检查 | 不执行恢复、重置或重启验证 |
| 7. ESP 备份 | 完整卷镜像及 149/149 文件校验通过 | 在线读取非原子 VSS 快照；备份仍在同一块 SSD 上 |
| 8. 条件性修复脚本 | 完成条件判断：当前不生成 BCDBoot 修复脚本 | 未发现支持 ESP/BCD 修复的证据，未执行任何修复 |

### 1. 硬件身份

`Win32_BIOS`：Manufacturer=`INSYDE Corp.`，SMBIOSBIOSVersion=`1.07.05COLO3`，ReleaseDate=`2025-08-14`。`Win32_BaseBoard`：Manufacturer=`COLORFUL`，Product=`P16 Pro`，Version=`Not Applicable`。`Win32_ComputerSystemProduct`：Vendor=`COLORFUL`，Name=`P16 Pro`，Version=`Not Applicable`；SystemSKUNumber 同为 `Not Applicable`。

BIOS SerialNumber、BaseBoard SerialNumber、Product IdentifyingNumber 三个字段均检出了 `V360…` 字样。完整编号仅保存在本地 `hardware.json`。这是身份字段中的线索，不等于已确认具体 Clevo 平台/子型号，也不提供固件包适用性依据。历史 `COL03` 与实际 `COLO3` 的 O/0 差异仍需保留。

当前 Windows 为 11 家庭中文版，`25H2 / 26200.9457`；最近启动时间为本日 `05:16:38 +08:00`。没有为了本次任务重启。

### 2. 物理磁盘、ESP 与固件启动项

当前仅枚举到 Disk 0，`YMTC PC41Q-1TB-B`，NVMe/GPT，容量 `1,024,209,543,168` 字节。

| 分区 | 类型 | 大小（字节） | 角色 |
|---|---|---:|---|
| Disk 0 Partition 1 | MSR | 134,217,728 | 保留分区 |
| Disk 0 Partition 2 | ESP | 104,857,600 | 唯一 EFI 系统分区，FAT32 |
| Disk 0 Partition 3 | Basic / C: | 578,254,364,672 | 当前 Windows |
| Disk 0 Partition 4 | Basic / D: | 429,495,681,024 | 数据卷 |
| Disk 0 Partition 5 | Recovery | 15,264,103,936 | WinRE 与 OEM 恢复映像 |

通过只读 UEFI API 读取并解析 `BootOrder`、`BootCurrent` 和对应 `Boot####`，通过 `QueryDosDevice` 核对卷与设备名。结果为：

```text
BootOrder   = 0001, 2001, 2002, 2003
BootCurrent = 0001
Boot0001    = Windows Boot Manager，ACTIVE
  GPT 分区签名 = 46cefd3f-da02-4a94-90ad-e10f228ef874
  分区号       = 2
  起始 LBA     = 264192
  分区扇区数   = 204800
  文件路径     = \EFI\MICROSOFT\BOOT\BOOTMGFW.EFI
Boot2001/2002/2003 = EFI USB / DVD-CDROM / Network
```

GPT 签名、LBA 与 `Get-Partition` 的 Disk 0 Partition 2 一致。该卷映射到 `\Device\HarddiskVolume2`；恢复卷映射到 `\Device\HarddiskVolume5`。`bcdedit /enum firmware /v` 和 `{bootmgr}` 的设备与路径亦一致。没有当前 Windows 实际从另一个 ESP 或备用文件启动的证据。

ESP 清单包含 49 个子目录、149 个文件，合计 `35,962,491` 字节，枚举错误为 0。顶层为 `EFI` 与 `System Volume Information`；EFI 下只有 `Microsoft` 和 `Boot`。检出了主 Boot Manager、备用 `bootx64.efi`、`bootmgr.efi`、内存检测、恢复工具、Microsoft Boot/Recovery 的两个 BCD Store。没有在该 ESP 发现 Linux、第三方引导器或独立 OEM EFI 目录。

主 BCD 的备份可由 BCDEdit 枚举，默认 Windows loader 为 `C:\Windows\System32\winload.efi`；resume 为同目录 `winresume.efi`；WinRE BCD 指向 Partition 5 的 `\Recovery\WindowsRE\winre.wim` 内 `\Windows\System32\winload.efi`。文件存在与配置一致。此结果不能单独证明所有 BCD 策略正确，但没有发现要重建 BCD 的具体异常。

### 3. EFI 文件、签名与版本

以下缩写仅用于表格：`ESP:` 为上述实际 EFI 卷；`Windows:` 为 `C:\Windows`；`WinRE:` 为上述 `winre.wim` 的 image 1。每一行完整路径、精确 UTC 修改时间、证书及哈希保存在本地 `efi-file-analysis.json`。

| 文件 | 大小（字节） | PE 固定文件版本 | 签发 CA | 整文件哈希编号 |
|---|---:|---|---|---|
| `ESP:\EFI\Microsoft\Boot\bootmgfw.efi` | 3,087,200 | 10.0.28000.367 | Windows UEFI CA 2023 | A |
| `ESP:\EFI\Boot\bootx64.efi` | 3,087,200 | 10.0.28000.367 | Windows UEFI CA 2023 | A |
| `ESP:\EFI\Microsoft\Boot\bootmgr.efi` | 3,070,456 | 10.0.28000.367 | Windows Production PCA 2011 | C |
| `ESP:\EFI\Microsoft\Boot\memtest.efi` | 2,610,656 | 10.0.26100.9444 | Windows Production PCA 2011 | D |
| `ESP:\EFI\Microsoft\Boot\SecureBootRecovery.efi` | 174,584 | 未检出固定版本资源 | Windows Production PCA 2011 | E |
| `Windows:\Boot\EFI\bootmgfw.efi` | 3,087,400 | 10.0.28000.367 | Windows Production PCA 2011 | B |
| `Windows:\Boot\EFI\bootmgr.efi` | 3,070,456 | 10.0.28000.367 | Windows Production PCA 2011 | C |
| `Windows:\Boot\EFI\memtest.efi` | 2,610,656 | 10.0.26100.9444 | Windows Production PCA 2011 | D |
| `Windows:\Boot\EFI\SecureBootRecovery.efi` | 174,584 | 未检出固定版本资源 | Windows Production PCA 2011 | E |
| `Windows:\Boot\EFI_EX\bootmgfw_EX.efi` | 3,087,200 | 10.0.28000.367 | Windows UEFI CA 2023 | A |
| `Windows:\System32\winload.efi` | 3,358,408 | 10.0.26100.9444 | Windows Production PCA 2011 | F |
| `Windows:\System32\winresume.efi` | 2,829,496 | 10.0.26100.9444 | Windows Production PCA 2011 | G |
| `WinRE:\Windows\System32\winload.efi` | 3,358,408 | 与 F 字节相同 | Windows Production PCA 2011 | F |

以上 13 个路径的 Windows Authenticode 均为 `Valid`，签名者为 Microsoft Windows。遍历 PE 的全部 `WIN_CERTIFICATE` 和 CMS 嵌套签名属性，每个文件检到 1 个主签名，没有检到额外嵌套主签名。使用 .NET SignedCms 验证密码学签名，再将自行计算的 PE/COFF Authenticode 摘要与已签署的 SPC 摘要比对，13/13 均通过；不是把整文件 SHA-256 当作 dbx 映像哈希。

实际 `bootmgfw.efi`、备用 `bootx64.efi` 和 `EFI_EX\bootmgfw_EX.efi` 整文件完全相同。Windows 的 2011 签名 `bootmgfw.efi` 虽然整文件哈希、大小不同，但 PE Authenticode SHA-256 与上述 2023 签名文件相同：

```text
F235CD21C919E05DA1218E8D13424F4BF2FEFFBB818E080666F93997FE26823A
```

这支持差异主要在签名封装等不计入该摘要的区域，不能据此诊断系统混装；不能直接将 Windows 普通 EFI 副本覆盖实际 2023 启动文件。

发现 Windows `FileVersionInfo.FileVersion` 显示文本与文件内部固定版本不一致：主 Boot Manager 显示 `28000.342`，备用显示 `28000.367`，但两个文件整文件哈希相同；内存检测/加载器显示旧的 `26100.1` 或 `26100.8875`，内部固定版本为 `26100.9444`。因此另读 PE `.rsrc` 中的 `VS_FIXEDFILEINFO` 并保留两种结果，以上表格使用直接读取的固定版本。资源/MUI 展示差异本身不是损坏或混装证据。ESP 上这些 EFI 文件修改时间主要为 `2026-09-13 03:02:02 UTC`，恢复工具为同日 `03:02:00 UTC`；精确时间见原始记录。

整文件 SHA-256 对照（文件身份/备份用途）：

```text
A 25D8869E3C79083AE62C698A9DB9030748E3267BD53593357447F3F72761D1F2
B 4BB32C300DF0815D8B6C54A0D3793AE1D78FCDC07F8E2B9F831289B2EB109092
C 0D2A3BE5DE6A9ABFD7E7E79D7A9D6974965CB1BE4EB4B975E32272F67B144C33
D 48454FACDAA88EBA041170323DC2A09C9845F9A758368D06817811C39687A7FB
E 8851B8D104841ED1AB75F1A1B60410FED078813936B03D6BDA8B2AC0044F4C1B
F 1C1A978448B8E0FC2284C13D6E55FDE8BF9827DCA432CDC8A9BE66C2E729ADDA
G 2778889F6151155B06D689AD827A4510342536FE998C0BEEFBC8CE63F3F743A3
```

### 4. 固件数据库完整解析

| 变量 | 读取 | 字节数 | EFI_SIGNATURE_LIST | 条目 |
|---|---|---:|---:|---|
| PK | 成功 | 844 | 1 | 1 张 X.509：Notebook Certificate |
| KEK | 成功 | 3,066 | 2 | 2 张 X.509：Microsoft KEK CA 2011、KEK 2K CA 2023 |
| db | 成功 | 9,310 | 7 | 7 张 X.509，详见下文 |
| dbx | 成功 | 19,996 | 1 | 416 条 SHA-256 数据 |
| SetupMode | 成功 | 1 | 不适用 | 原始字节 `00` |
| SecureBoot | 成功 | 1 | 不适用 | 原始字节 `00` |

`Confirm-SecureBootUEFI=False` 是成功执行后的值，与当前 SecureBoot 关闭一致。SetupMode 为 0、PK 已安装，不是“没有平台密钥”的状态。四个密钥数据库属性均为 Non Volatile / BootService Access / Runtime Access / Time Based Authenticated Write Access；状态变量只有 BootService / Runtime Access。

逐列表核对长度、签名条目尺寸、边界并解析 GUID/X.509；四个数据库未解析字节均为 0，无未知类型。db 的七张证书为：Microsoft Windows Production PCA 2011、Windows UEFI CA 2023、Microsoft Corporation UEFI CA 2011、Microsoft UEFI CA 2023、Microsoft Option ROM UEFI CA 2023、Secure Certificate、Cus CA。后两张证书的来源/出厂归属未确认；名称本身不构成异常或恶意证据。

Windows UEFI CA 2023 证书 DER 的 SHA-256 为：

```text
076F1FEA90AC29155EBF77C17682F75F1FDD1BE196DA302DC8461E350A9AE330
```

实际 Boot Manager 的 signer 到该 db 信任锚之间的证书签名由公钥验证通过；所有本轮已读 EFI signer 均可连接到相应 db CA。此处只验证证书签名关系，不把 Windows 在线信任策略当成 OEM 固件策略，也不要求该 db CA 必须是自签名根。

当前 dbx 中没有 X.509 完整证书或 `EFI_CERT_X509_SHA256/SHA384/SHA512` 的 TBS 哈希撤销条目，只有 SHA-256 类型。将本轮所有相关 PE Authenticode 映像摘要与 dbx 条目逐项比对，命中数为 0。不能用“搜索不到 2011 名称”替代这一结果，也不能据此排除数据库之外的固件行为、其他策略或尚未检查组件。[UEFI 规范](https://uefi.org/specs/UEFI/2.10/32_Secure_Boot_and_Driver_Signing.html?highlight=authenticated+variable)及[微软 PE 格式文档](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)说明相关格式与摘要规则。

### 5. Windows 更新历史、任务与 SVN

管理员查询任务 `\Microsoft\Windows\PI\Secure-Boot-Update`：存在，Enabled=True，State=Ready，UserId=SYSTEM；LastRunTime=`2026-10-04 05:21:57 +08:00`，LastTaskResult=`0`，ActionData=`SBServicing`。没有运行、修复或重新创建此任务。任务存在与返回 0 不单独证明固件能启动；第一阶段“未找到任务”的非管理员观察已经被本轮现状更新。

只读注册表结果：`UEFISecureBootEnabled=0`，`UEFICA2023Status=Updated`，`AvailableUpdates=0`，`WindowsUEFICA2023Capable=2`；Servicing key 的值名中没有 `UEFICA2023Error`、`UEFICA2023ErrorEvent`。其他原始状态如 `TaskState=13`、`LastFirmwareFreezeStateAtTaskStart=13` 已保存，未在缺少明确官方定义时推断成故障。状态值不能代替实际固件变量。

System 当前保留 12,928 条事件，其中读取 TPM-WMI provider 的全部现存 284 条；日期范围 `2025-04-07` 至 `2026-10-04 09:28:32 +08:00`。含 21 条 1796，最近仍为 `2026-08-15 07:59:40 +08:00`，XML 只有 `HResult=-2147024809`（`0x80070057`），未提供明确更新类型；这仍是历史线索。另有 2025-10-08 的 1034 和 2026-08-12 的两条 1800；本范围没有 1037、1042、1043、1044、1045、1799、1803 或 1808。无事件不等于从未更新，事件也未与当前故障建立因果关系。[微软事件说明](https://support.microsoft.com/en-us/servicing/os/windows/2022/06/secure-boot-db-and-dbx-variable-update-events)用于区分更新类别。

还枚举日志与 provider，并读取 Kernel-Boot/Operational 最近 30 天的最多 100 条及 WindowsUpdateClient/Operational 最近 90 天的现存 715 条；后者消息中未检出 Secure Boot/安全启动字样。本机未注册名称匹配 `*Secure*Boot*` 的独立 provider，相关信息由 TPM-WMI 的 System 事件提供。最近可读补丁为 2026-09-29 的 KB5129195、09-18 的 KB5126052、09-14 的 KB5124007；这里只记录本机安装信息，不据此保证更新完整性。

SVN 三种状态已区分：

| 状态 | 实际读取 |
|---|---|
| 启动文件内 | 主/备用及 Windows 对照 Boot Manager 的 `BOOTMGRSECURITYVERSIONNUMBER` 资源为 `00000B00`，解析为 11.0 |
| 本地待应用载荷 | `C:\Windows\System32\SecureBootUpdates\DBXUpdateSVN.bin` 中 Boot Manager SVN 为 11.0 |
| 已读取的固件 dbx | 未检出对应 `EFI_BOOTMGR_DBXSVN_GUID`（9d132b61-59d5-4388-ab1c-185c3cb2eb92）的记录，匹配数为 0 |

因此不能把“待应用载荷存在”写成“固件 SVN 已更新”，目前没有读到固件 SVN 高于此 Boot Manager、导致其自拒绝的证据。记录格式参考[微软公开 Secure Boot 对象](https://github.com/microsoft/secureboot_objects/blob/9b3f8df2e1774d18ad4df8e36fcd91781fb0251a/PreSignedObjects/DBX/dbx_info_msft_latest.json)，没有执行其中脚本；SVN 的行为依据[微软 KB5025885](https://support.microsoft.com/en-us/servicing/os/windows/2023/03/how-to-manage-the-windows-boot-manager-revocations-for-secure-boot-changes-associated-with-cve-2023)。对公开仓库的引用只用于格式核对，不把其版本代替本机状态。

### 6. WinRE 与 OEM 恢复来源

`reagentc /info` 成功：WinRE Enabled，配置位置为 Disk 0 Partition 5 的 `\Recovery\WindowsRE`，BCD identifier=`8dfc21b9-a43b-11f0-adbb-fbd1f4661475`，版本 `10.0.26100.9444`。BCD 的 ramdisk WIM 和 SDI 设备均映射到该分区。

`winre.wim`（864,059,682 字节）与 `boot.sdi`（3,170,304 字节）存在且可读取；WIM image 1 元数据读取成功，为 amd64 WindowsPE，语言 en-US/zh-CN，版本 `26100.9444`。使用 WIMGAPI 以只读源句柄、单文件提取方式读取 WIM 中 BCD 实际引用的 `Windows\System32\winload.efi`，没有挂载或应用映像；文件与当前系统的加载器整文件 SHA-256 相同，签名和摘要检查通过。配置正确和文件可读取，不等于已经验证 WinRE 在 Secure Boot 下能启动。

恢复映像配置还指向 `\RecoveryImage`，index=1；该目录实际包含 `install.wim`（12,972,616,040 字节）。元数据读取成功，名称 CaptureImage，版本 `10.0.26100.2314`，EditionId=CoreCountrySpecific，zh-CN。存在 `C:\Recovery\OEM` 的 ResetConfig、恢复后脚本、备份目录；CustomTool.xml 显示“COLORFUL 一键还原 / COLORFUL 一键恢复出厂设置”。ResetConfig 配置了 BasicReset/FactoryReset 的 AfterImageApply 脚本，该脚本含对 RecoveryImage 的查找及恢复映像重新登记。只读取文本，没有运行任何 OEM 程序或脚本。

这证明存在 OEM 恢复资源与菜单配置，并不保证每个“重置此电脑”入口一定还原这份旧映像。微软说明：本地重装使用设备上的文件，云下载下载 Windows 副本；Windows 10/11 的标准重置支持无需单独恢复映像的流程，OEM 定制与入口也可能影响结果。不能把本地重装、云下载与 COLORFUL 一键还原视为同一种恢复路径，或仅因恢复分区存在就断言恢复出厂系统。[重置选项说明](https://support.microsoft.com/en-us/windows/experience/backup-recovery/reset-your-pc)、[Push-button reset](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/push-button-reset-overview?view=windows-11)。本轮不建议以重装或重置替代未确定根因的排查。

### 7. 已完成的 ESP 备份

原样备份位于上述本地证据目录：

```text
ESP-Backups\Disk0-Partition2.img
ESP-Backups\Disk0-Partition2-Verified\
verified-esp-file-manifest.json
backup-verified-summary.json
```

原始镜像通过只读卷句柄读取唯一 ESP 的全部 `104,857,600` 字节。两次完整源读取与镜像 SHA-256 均为：

```text
6123853207F279F23092715ED3A1A6FC749A66EF32E3EB9FD628A753271425D3
```

从已校验镜像按 FAT32 文件链读取全部 149 个文件到新建的 `Disk0-Partition2-Verified`，恢复 49 个目录，记录原始目录清单、大小、修改时间及源/目标哈希，149/149 校验通过，0 失败；备份文件总计 `35,962,491` 字节。再次核对镜像哈希仍一致，最终原样目录没有用 BCDEdit 打开。

保留了备份过程中的真实限制：最初普通文件读取被系统锁定的 `BCD`、`BCD.LOG` 阻止，147/149 通过，不能称其完整。随后从原始镜像提取两文件，用 `bcdedit /store` 枚举**仓库外的工作副本**时，观察到该工作副本的 BCD/日志哈希改变。因此保留早期工作副本和错误，不覆盖它；另建上面的 Verified 目录，全部重新从原始镜像提取校验。`ESP-Backups\Disk0-Partition2\` 是分析副本，不能再当作原样备份。

镜像采集约在 09:56:29，在线读取不是原子 VSS 快照，但连续两次源读取一致；后续源变化不能倒推为镜像校验失败。备份写入 C 盘，位于 ESP 与仓库之外，没有改写 ESP；仍与 ESP 位于同一物理 SSD，不提供硬件故障隔离。

### 8. 不生成 BCDBoot 修复脚本的依据

当前 BootCurrent 与 Boot0001 指向唯一 ESP 的实际 Windows Boot Manager，GPT/LBA/设备名一致；BCD 默认加载器及 WinRE 路径可解析且文件存在；实际 EFI 文件摘要与签名验证通过，主文件和 Windows `EFI_EX` 副本整文件相同，未命中当前 dbx。尚未发现文件缺失、签名破坏、BCD 无法枚举或启动项目标错误等支持重建的证据。

因此按 `TEMP-TASK.md` 第八项的条件，本轮完成“不生成修复脚本”的判断；没有为了生成而先运行 BCDBoot，也没有将 `/bootex` 当作试错参数。若以后证据转向 ESP/BCD，须另核对届时本机工具版本/帮助、微软文档、源文件签名与具体变化，再生成默认只展示计划的审查脚本。当前已有备份不等于授权修复。

### 读取错误、工作假设与未完成验证

本机读取错误保留在本地证据中：BootNext 的只读 API 返回 Windows 错误 `203`（未取得值，不转换成 False）；普通 BCD/BCD.LOG 文件读取被占用（由只读镜像备份解决）；恢复分区 `System Volume Information` 枚举拒绝访问，其他已配置恢复目录可读；WinRE 中额外尝试的 `winresume.efi` 返回找不到指定路径，它不是当前 WinRE BCD 引用文件，不据此判定 WinRE 损坏。独立 Secure Boot provider 名称查询也保留“没有匹配”的错误，日志来源以实际 TPM-WMI/System 为准。

早期临时工具存在一次动态 C# 类型引用编译错误、一次 PowerShell 管道语法错误，修正后重跑相应检查成功；它们不是固件读取错误，也不用于判断证书缺失。

当前工作假设：常见的单纯缺少 CA、所检查启动映像命中 dbx、ESP/BCD 目标缺失均不获本轮证据支持。若历史故障仍可复现，固件实际验证/配置与尚未检查的启动策略或组件值得继续检查；这不是确认 BIOS 损坏。

未完成且不能靠静态读取替代的项目：

1. 向用户确认历史开启 enforcement 失败是否仍复现，以及其间是否改过固件设置。该事实尚未获得本轮回传。
2. 当前 Secure Boot 关闭时读到的固件变量，不能保证开启后内容与执行策略完全相同；未进行开启、重启或启动对照实验。
3. 完整后续驱动、代码完整性策略和 OEM 专有验证流程未穷尽；不能将 13 个 EFI 文件的检查称为整个 Windows 启动链已获固件验证。
4. `V360…` 的完整平台匹配和适用的 OEM 固件包尚未核实；没有刷写、下载并执行更新包或联系第三方。
5. WinRE、标准重置和 OEM 一键恢复未实启动；OEM 旧 WIM 的存在不证明实际恢复选择或成功率。

本阶段未执行 TPM 清除、密钥删除/重置、固件写入、EFI 文件修复、注册表/任务修改、BCDBoot 修复、BIOS 更新、重装、格式化或自动重启；未提交/推送原始数据或改变仓库可见性。

结束复核：重新读取六个固件变量并比对原始 SHA-256，全部内容未变；重新核对实际 ESP 的五个 EFI 文件整文件哈希，全部未变。最终原样备份 149/149 通过，修改的五份项目文档已检查相对链接，未包含本轮读取的完整硬件序列号/SMBIOS UUID。复核原始结果保存在本地 `post-check-source-state.json`。
