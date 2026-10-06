# COLORFUL 本机管理员只读复查（2026-10-07）

记录日期：2026-10-07（Asia/Hong_Kong，UTC+08:00）。**问题未解决，根因未确定。本页只报告新的实测与证据边界，不排列原因。**

## 来源、授权与配置阶段

- **[用户反馈]** 用户确认当前连接的就是受影响本机，并表示关闭安全启动后“随便启动”。记录为关闭时能正常启动的最新反馈，不扩大为所有介质已逐一实测。
- **[本机实测]** Codex 初始进程不具备管理员权限；读取机型、BIOS、Windows 状态与可访问文件。用户随后明确授权由 Codex 自行运行已准备的只读采集脚本。
- **[管理员实测]** Codex 通过 Windows 管理员进程运行采集，报告结束时间为 `2026-10-07T04:02:56.6731208+08:00`；采集内再次确认 Administrator=True。不是另一台 ASUS 台式机的 USB 检查。
- 采集对象是**本次安全启动关闭状态**的 Windows、固件变量和当前 ESP 文件。没有重新开启安全启动、重启或实际启动安装介质，没有再次清除／恢复／追加密钥。
- 原始变量、文件副本和详细 JSON 留在本地新证据目录；提交 GitHub 的是经过整理的文档，不包含恢复密钥、硬件序列、完整设备 UUID、原始卷标识或未经审查的系统数据。

本页是 [10-06 BIOS 现场记录](INVESTIGATION-2026-10-06.md)之后的新快照。此前清除后是否为空、恢复操作前后是否完全一致，仍不能用本次结果倒推。

## 1. 身份、关闭状态与分区差异

| 检查 | 本次结果 | 来源／限制 |
|---|---|---|
| 整机 | COLORFUL P16 Pro | 本机 CIM |
| BIOS | INSYDE Corp.；`1.07.05COLO3`；日期 2025-08-14 | 本机 CIM；没有更新 BIOS |
| Windows | Windows 11 家庭中文版；构建 26300 | 管理员采集；采集前注册表另读到 26H2、UBR=9457，即 26300.9457 |
| 安全启动 | `Confirm-SecureBootUEFI=False`；`SecureBoot=00` | 管理员命令成功后的值，区别于最初访问拒绝 |
| 设置模式 | `SetupMode=00` | 本次管理员实测；不是前次清除阶段的读取 |
| 当前 ESP | Disk 0 Partition 1，209,715,200 字节（200 MiB），唯一可枚举 ESP | 当前 Get-Partition；不是沿用旧分区号 |

当前可枚举布局为 ESP、MSR、C:、Recovery 共四个分区；10-04 报告为五个分区，ESP 位于 Partition 2、100 MiB，Windows 构建为 26200.9457。**布局与系统版本有差异，但本次没有确定变更发生的时间或原因，也没有核实旧备份是否仍可用。** 不默认旧原始 Boot0001、旧 ESP 路径或旧恢复分区仍对应现在。

历史 CPU、KBC/EC、ME 及准确子型号沿用历史来源，本轮没有重新测量这些字段，也没有核实适用 BIOS 包。

## 2. 当前与默认安全启动数据库

逐列表验证 `EFI_SIGNATURE_LIST` 的边界、头长、条目尺寸和 GUID，并解析 X.509／SHA256。本次成功读取的四个当前数据库及四个默认数据库都解析到文件末尾，无未解析尾部。

| 变量 | 字节数 | X.509 数 | SHA256 数 | 原始变量整段 SHA256 |
|---|---:|---:|---:|---|
| `PK` | 844 | 1 | 0 | `3BF0F2E0A51E4F6F2798DBC16954FFA91F041D5EC70505C28D3BEEE3D7A027AD` |
| `KEK` | 3066 | 2 | 0 | `CC3A5DBC7B3AEC3B60C0DA33510BF93F402479BBF445DC360E6111AFA70C6342` |
| `db` | 9674 | 7 | 7 | `6F68CD8B39C0F465AB1483E5A9B2AB8981F1CED49F1689570774448683A51927` |
| `dbx` | 19996 | 0 | 416 | `721B4FE1FD53DB325FD1783099026CE0B44272F4E4CA23E4B89C1046C0978A47` |
| `PKDefault` | 844 | 1 | 0 | `3BF0F2E0A51E4F6F2798DBC16954FFA91F041D5EC70505C28D3BEEE3D7A027AD` |
| `KEKDefault` | 3066 | 2 | 0 | `CC3A5DBC7B3AEC3B60C0DA33510BF93F402479BBF445DC360E6111AFA70C6342` |
| `dbDefault` | 9310 | 7 | 0 | `3D7964102AAA73317CF2EA06944581422652277DAD1B52EC8414EDD92CCD0359` |
| `dbxDefault` | 19996 | 0 | 416 | `721B4FE1FD53DB325FD1783099026CE0B44272F4E4CA23E4B89C1046C0978A47` |

当前数据库属性为 Non Volatile、BootService Access、Runtime Access、Time Based Authenticated Write Access；默认数据库与两个状态变量返回 BootService Access、Runtime Access。

### 与本次提供的默认变量比较

- `PK == PKDefault`、`KEK == KEKDefault`、`dbx == dbxDefault`：本次原始字节完全相同。
- `db` 的前 9310 字节与 `dbDefault` 完全相同，尾部增加一个 364 字节的 SHA256 签名列表，含 7 条摘要；当前 db 合计 9674 字节。
- db 和 dbDefault 的 7 张证书 DER 一致，其中 Windows UEFI CA 2023 的指纹也与 10-04 已公布值一致。
- 当前 dbx 只包含 416 条 SHA256，没有本次解析到的完整 X.509 或其他类型条目。

**默认变量比较只证明本次这几个变量的字节关系，不证明整个固件恢复为出厂状态，也不替代前次清除／恢复操作的前后快照。** 没有复核全部 OEM 策略、DBR 或固件执行行为。

### db 中证书的 DER SHA256 指纹

下表是本次实际解析的证书内容，区别于前次照片中仅可见的名称。

| 证书 CN | DER SHA256 |
|---|---|
| `Microsoft Windows Production PCA 2011` | `E8E95F0733A55E8BAD7BE0A1413EE23C51FCEA64B3C8FA6A786935FDDCC71961` |
| `Windows UEFI CA 2023` | `076F1FEA90AC29155EBF77C17682F75F1FDD1BE196DA302DC8461E350A9AE330` |
| `Microsoft Corporation UEFI CA 2011` | `48E99B991F57FC52F76149599BFF0A58C47154229B9F8D603AC40D3500248507` |
| `Microsoft UEFI CA 2023` | `F6124E34125BEE3FE6D79A574EAA7B91C0E7BD9D929C1A321178EFD611DAD901` |
| `Microsoft Option ROM UEFI CA 2023` | `E5BE3E64C6E66A281457ECDECE0D6D0787577AAD2A3A0144262C10C14BA8D8F1` |
| `Secure Certificate` | `7373145CFFBDC58304B34EC0DBC9176D02788D167CF02F44BF397AD19C785176` |
| `Cus CA` | `40EBF95EA377CB04E6DD3358A97B7C62B1C6E15628E53C40BEE1DAE0616132E0` |

## 3. 当前 BCD 目标与 ESP 文件

`bcdedit /enum firmware /v`、`/enum {bootmgr} /v`、`/enum {current} /v` 均成功，退出码 0：

- Windows Boot Manager 的 device 为 `partition=\Device\HarddiskVolume1`，path 为 `\EFI\Microsoft\Boot\bootmgfw.efi`。
- 当前加载器指向 C: 的 `\WINDOWS\system32\winload.efi`，osdevice=C:，systemroot=\WINDOWS。
- QueryDosDevice 将本次 ESP 已有卷路径映射为 `\Device\HarddiskVolume1`，与 BCD 指向对应；完整卷标识仅保留在本地。
- 本次没有读取原始 `BootCurrent`、`BootOrder` 或 `Boot####` 的二进制设备路径，不声称已复现 10-04 那组完整固件路径核对。

通过 ESP 已有卷路径读取源文件，没有分配盘符或挂载操作；写入新建证据目录的是副本。每份副本的整文件 SHA256 与当次源字节的哈希一致。

### ESP 文件的整文件 SHA256

| 路径（相对当前 ESP） | 字节数 | 整文件 SHA256 |
|---|---:|---|
| `EFI\Boot\bootx64.efi` | 3087200 | `25D8869E3C79083AE62C698A9DB9030748E3267BD53593357447F3F72761D1F2` |
| `EFI\Microsoft\Boot\bootmgfw.efi` | 3087200 | `25D8869E3C79083AE62C698A9DB9030748E3267BD53593357447F3F72761D1F2` |
| `EFI\Microsoft\Boot\bootmgr.efi` | 3070456 | `0D2A3BE5DE6A9ABFD7E7E79D7A9D6974965CB1BE4EB4B975E32272F67B144C33` |
| `EFI\Microsoft\Boot\memtest.efi` | 2610656 | `48454FACDAA88EBA041170323DC2A09C9845F9A758368D06817811C39687A7FB` |
| `EFI\Microsoft\Boot\SecureBootRecovery.efi` | 174584 | `8851B8D104841ED1AB75F1A1B60410FED078813936B03D6BDA8B2AC0044F4C1B` |

`bootmgfw.efi` 与备用 `bootx64.efi` 的整文件 SHA256 相同，也与当前 Windows `Boot\EFI_EX\bootmgfw_EX.efi` 副本相同。普通 Windows `Boot\EFI\bootmgfw.efi` 为另一个整文件值，见下表；两者 PE 映像摘要相同。这些值也与 10-04 报告相应已公布整文件值一致，不因此恢复旧分区身份。

### Windows 副本与加载器

| 路径（相对 Windows） | 整文件 SHA256 | PE 摘要匹配当前 db 哈希项 | 匹配当前 dbx 哈希项 |
|---|---|---|---|
| `Boot\EFI\bootmgfw.efi` | `4BB32C300DF0815D8B6C54A0D3793AE1D78FCDC07F8E2B9F831289B2EB109092` | 是 | 否 |
| `Boot\EFI\bootmgr.efi` | `0D2A3BE5DE6A9ABFD7E7E79D7A9D6974965CB1BE4EB4B975E32272F67B144C33` | 是 | 否 |
| `Boot\EFI_EX\bootmgfw_EX.efi` | `25D8869E3C79083AE62C698A9DB9030748E3267BD53593357447F3F72761D1F2` | 是 | 否 |
| `System32\winload.efi` | `1C1A978448B8E0FC2284C13D6E55FDE8BF9827DCA432CDC8A9BE66C2E729ADDA` | 否 | 否 |
| `System32\winresume.efi` | `2778889F6151155B06D689AD827A4510342536FE998C0BEEFBC8CE63F3F743A3` | 否 | 否 |

加载器未匹配“单文件哈希允许项”不等于不受信任：本次它们的内嵌签名者可由 db 中的对应 CA 签名验证，见下一节。这里不将证书信任与哈希登记混为同一检查。

## 4. 内嵌签名、PE 摘要与允许／撤销条目

对本次保存的 10 个 EFI 路径：

1. 遍历 PE Certificate Table 和 WIN_CERTIFICATE，解码 PKCS#7，检查嵌套签名属性；10 个路径均为一份主内嵌签名，未发现嵌套签名。
2. 对 CMS 运行仅验证签名的 `SignedCms.CheckSignature(true)`，均成功。
3. 独立复算 SHA256 PE/COFF 映像摘要，与签名内容中携带的摘要比较，均相同。
4. 对签名者与当前 db 中对应 CA 的直接证书签名关系进行公钥验证，均通过。这不是在线 Windows 信任策略检查，也不是 OEM 固件运行时策略模拟。
5. 将相应 PE 摘要与本次 db/dbx 的 SHA256 条目比较，结果如下。

ESP 主／备用启动器以及 Windows EFI_EX 副本的内嵌签发者是 **Windows UEFI CA 2023**；其他所查路径为 Microsoft Windows Production PCA 2011。本机 `Get-AuthenticodeSignature` 返回 `SignatureType=Catalog`，其显示的 2011 签发者不能代替上述内嵌签名遍历结果。

| ESP 文件 | 本次计算的 PE/COFF SHA256 | 当前 db 哈希匹配 | 当前 dbx 哈希匹配 |
|---|---|---|---|
| `EFI\Boot\bootx64.efi` | `F235CD21C919E05DA1218E8D13424F4BF2FEFFBB818E080666F93997FE26823A` | 是 | 否 |
| `EFI\Microsoft\Boot\bootmgfw.efi` | `F235CD21C919E05DA1218E8D13424F4BF2FEFFBB818E080666F93997FE26823A` | 是 | 否 |
| `EFI\Microsoft\Boot\bootmgr.efi` | `9BF39341F9161A83D73D4FCCBF3FB68C588974E88550E30D4DBC8CB518EAECB7` | 是 | 否 |
| `EFI\Microsoft\Boot\memtest.efi` | `C7EEF00B4FF57AD23FA82C59FCEAC446ED9ABB0703BA6FB9D67A3B3AC8AEA875` | 是 | 否 |
| `EFI\Microsoft\Boot\SecureBootRecovery.efi` | `2DEF2CC16BBC9A75654E6D14430FFBA25710C17BF1F85B0CA4213430ECD46858` | 是 | 否 |

这次直接取得了主启动器完整 PE 摘要 `F235CD21C919E05DA1218E8D13424F4BF2FEFFBB818E080666F93997FE26823A`，并与新读取的 db 完整条目匹配。**这是新测量补上了完整匹配证据，不是补全或替换 10-06 截图里截断的旧值。**

### 当前 7 条允许哈希的完整对应

下列行号是本次二进制列表解析顺序，不擅自对应为 BIOS 画面的 08～14 行号。

| 解析序号 | db 中完整 SHA256 | 本次独立对应的 ESP 文件 |
|---|---|---|
| 1 | `F235CD21C919E05DA1218E8D13424F4BF2FEFFBB818E080666F93997FE26823A` | `EFI\Boot\bootx64.efi`、`EFI\Microsoft\Boot\bootmgfw.efi` |
| 2 | `9BF39341F9161A83D73D4FCCBF3FB68C588974E88550E30D4DBC8CB518EAECB7` | `EFI\Microsoft\Boot\bootmgr.efi` |
| 3 | `C7EEF00B4FF57AD23FA82C59FCEAC446ED9ABB0703BA6FB9D67A3B3AC8AEA875` | `EFI\Microsoft\Boot\memtest.efi` |
| 4 | `2DEF2CC16BBC9A75654E6D14430FFBA25710C17BF1F85B0CA4213430ECD46858` | `EFI\Microsoft\Boot\SecureBootRecovery.efi` |
| 5 | `7416ABCD3C9627450BEECD01710490674572C34062025FB4D67337E67324A213` | 尚未独立对应到当前文件 |
| 6 | `4772C4063917AE3D110A34041DE235E911D8D26DEAA06644500B86524D48C4B5` | 尚未独立对应到当前文件 |
| 7 | `1F6656777197F390F87412404AB12B4B4BD3339DDCE31316266376829CD256F3` | 尚未独立对应到当前文件 |

当前 5 个 ESP 路径对应 4 种已匹配的映像摘要；另 3 条允许摘要尚未独立映射，不能仅凭旧反馈断言它们就是 3 个特定 .mui 文件。没有因为“仍失败”而继续追加任何条目。

## 5. 状态、读取错误与方法限制

- 初始非管理员 `Confirm-SecureBootUEFI` 返回“无法设置正确的权限。访问被拒绝。”、`SetPrivilegeFailed`；后续管理员采集成功返回 False，分别保留，不把读取失败当成关闭值。
- 管理员 `Get-SecureBootUEFI -Name dbt` 和 `dbtDefault` 均返回“当前未定义变量: 0xC0000100”。不扩展为全部 OEM 数据库或所有策略不存在。
- 注册表当前读取到根键 AvailableUpdates=0、State 的 UEFISecureBootEnabled=0、Servicing 的 UEFICA2023Status=Updated。Servicing 的 UEFICA2023Error／UEFICA2023ErrorEvent 未取得；根键 WindowsUEFICA2023Capable 未取得，未遍历其他路径寻找同名值。
- Secure-Boot-Update 任务存在、启用、Ready，LastRunTime 为 2026-10-07 03:20:37 +08:00，LastTaskResult=0。任务结果不证明安全启动时可以启动。
- 非管理员源路径 VersionInfo 与管理员采集副本的 FileVersionInfo 曾分别显示 Boot Manager 28000.342／28000.367；整文件 SHA256 一致。本次没有解释该字段差异，不将其写成文件发生变化。
- X.509 解析库报告了关于非正证书序列号的兼容性警告；当次解析仍完成。没有据此判断证书就是故障原因。
- 复算器对所查文件验证了头部、节表边界、连续的原始节布局和文件末尾证书表，才计算摘要；结果与签名所含摘要逐一匹配。方法与 [微软 PE 格式](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)、[EDK II 的 HashPeImage 实现](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/DxeImageVerificationLib.c)对照。EDK II 用作算法参考，不代表本机 Insyde 固件源码。
- 变量读取及权限说明参考 [微软 Get-SecureBootUEFI 文档](https://learn.microsoft.com/en-us/powershell/module/secureboot/get-securebootuefi)。本轮没有重新测量全部 SVN、后续驱动、WinRE 或所有代码完整性策略，也没有重查旧 USB。

## 6. 补齐的证据与仍未完成的事项

这次补齐了**当前关闭状态**的原始数据库采集、当前默认变量比较、当前 ESP／BCD 目标对应、所查内嵌签名与完整 PE 摘要验证，以及 4 种允许哈希与实际文件的对应。原先“新增条目是否匹配所选启动器”的部分缺口已有新证据。

仍没有：

- 新的 Secure Boot ON 启动测试、准确失败返回码或执行跟踪；不能判断固件是否已放行、失败在启动器前还是后。
- 前次清除阶段的空库证明、恢复操作前后的完整快照、全部 OEM 状态对照。
- 其余 3 条允许哈希的当前文件对应或整个启动链覆盖证明。
- 当前原始 BootCurrent／Boot#### 设备路径，以及完整后续策略、驱动和撤销规则的穷尽检查。
- 已核实介质身份与内容连续一致的旧 USB 关联，或明确的 2011／2023 安装介质启动配对。
- 适用 BIOS 包的官方核实，或任何新的修复／刷写／重装结果。

**静态校验通过和 db 完整匹配，不等于固件在 ON 状态实际执行放行；故障根因仍未确定。** 不把这次关闭状态的读取描述为已修好，或描述成重新拍摄了上次失败画面。

## 本地证据位置与发布范围

本聊天 `outputs/SecureBootEvidence-20261007-040246-ba72b6/` 中保存 `report.json`、`firmware/`、`files/`、`embedded-signatures.json`、`analysis.json` 和 `volume-mapping.json`；本地另有采集与分析代码、中文结果摘要。原始证据和脚本未在本次上传 GitHub，也没有覆盖 10-04 的旧备份。

本轮只读取系统与固件源数据，写入新建的本地证据目录及文档草稿。未执行密钥写入、EFI 修改、BCD 修复、任务或注册表修改、TPM 清除、恢复程序、格式化或重启。用户随后明确要求将本次补充提交 GitHub；这不自动授权下一轮电脑修复。
