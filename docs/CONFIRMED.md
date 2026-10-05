# 已确认事实

更新日期：2026-10-05

本文件只放亲自观察到或截图能够直接支持的事实；推测放到其他文档。

## 2026-10-05 现场复测新增确认

- 用户已完成一次新的 Windows 安装；随后再次开启 `Enforce Secure Boot`，内部 Windows Boot Manager 仍启动失败。
- `Enforce Secure Boot = Disabled` 时，内部 Windows 可正常启动。
- 保存 `Enforce Secure Boot = Enabled` 并重启后，内部 Windows Boot Manager 仍出现 boot failed。
- Boot Manager 中能同时看到内部 `Windows Boot Manager (... YMTC PC4...)` 与 `EFI USB Device (VendorCoProductCode)`。
- 在 Secure Boot enforcement 开启状态下手动选择该 USB，固件直接显示 `EFI USB Device (VendorCoProductCode) boot failed.`。
- 用户确认该 USB 在 enforcement 关闭时可用于正常启动/安装 Windows。
- 因此已现场形成新的 A/B：**Secure Boot OFF 时 Windows/USB 均可启动；Secure Boot ON 时 Windows/USB 均被拒绝。**
- BIOS `DB Options` 现场直接显示 7 个 PKCS7 条目：
  1. `Microsoft Windows Production PCA 2011`
  2. `Windows UEFI CA 2023`
  3. `Microsoft Corporation UEFI CA 2011`
  4. `Microsoft UEFI CA 2023`
  5. `Microsoft Option ROM UEFI CA 2023`
  6. `Secure Certificate`
  7. `Cus CA`
- `DBX Options` 现场显示多条 `[SHA256]` 撤销项并可滚动；本轮没有删除或新增任何 DBX 项。
- Secure Boot 管理页还可见 `KEK Options`、`DBT Options`、`DBR Options`。
- `Select a UEFI file as trusted for execution` 的页面说明为 `Add specific EFI image hash to allowed database.`；本轮没有执行该写入。
- 本轮没有执行 `Erase all Secure Boot Settings`、`Restore Secure Boot to Factory Settings`、手工删除/新增 PK/KEK/db/dbx，也没有刷 BIOS/EC。

完整的新现场复测和结论边界见 [INVESTIGATION-2026-10-05.md](INVESTIGATION-2026-10-05.md)。

## 2026-10-04 新增确认

- 本机 SMBIOS 型号为 COLORFUL P16 Pro；Windows 11 25H2，`26200.9457`；Disk 0 为 NVMe/GPT。
- 用户管理员 PowerShell 读取 db，搜索 `Windows UEFI CA 2023` 返回 `True`。
- db 中搜索 `Microsoft Windows Production PCA 2011` 返回 `True`；dbx 中同名搜索返回 `False`。这只记录名称搜索结果，未排除哈希撤销。
- 实际 EFI 分区 `\EFI\Microsoft\Boot\bootmgfw.efi` 的 Authenticode 为 `Valid`，签发者为 `Windows UEFI CA 2023`。
- `bcdedit /enum {bootmgr}` 指向 `\Device\HarddiskVolume2` 的上述文件；固件启动顺序将 Windows Boot Manager 放在 USB、光驱与网络之前。
- Windows 注册表记录 `UEFISecureBootEnabled=0`、`UEFICA2023Status=Updated`、`AvailableUpdates=0`。
- 第一阶段尚未取得 `Confirm-SecureBootUEFI`、`SetupMode`、PK 和 KEK 的管理员结果；第二阶段已由本机管理员读取补齐，见下文。

## 2026-10-04 管理员取证新增确认

- 执行进程的管理员 token 检查为 True；`Confirm-SecureBootUEFI=False`、`SecureBoot=00`、`SetupMode=00` 均来自成功读取。
- PK/KEK/db/dbx 分别为 844 / 3066 / 9310 / 19996 字节；完整 EFI_SIGNATURE_LIST 解析分别得到 1 / 2 / 7 张 X.509 和 416 条 SHA-256 数据，未解析字节均为 0。
- db 的实际 Windows UEFI CA 2023 DER 指纹已核对，Boot Manager 到该证书的签名关系验证通过；本轮 13 个相关 EFI 路径均通过 CMS 签名与 PE 摘要比对，未命中当前 dbx。
- 当前仅枚举到 1 块物理磁盘、5 个分区、1 个 ESP。固件 `BootCurrent=0001`；`Boot0001` 的 GPT 分区签名、LBA 与 Disk 0 Partition 2 一致，指向 `\EFI\MICROSOFT\BOOT\BOOTMGFW.EFI`；卷设备映射到 `\Device\HarddiskVolume2`。
- 实际 Boot Manager、备用 bootx64.efi、Windows EFI_EX 的 bootmgfw_EX.efi 整文件 SHA-256 相同，内部固定版本为 `10.0.28000.367`；普通 Windows EFI 副本为 2011 签名，但 PE Authenticode 摘要相同。Windows 版本显示文本与内部固定版本的差异已单独记录。
- Secure-Boot-Update 任务存在、Enabled=True、SYSTEM、Ready，2026-10-04 05:21:57 +08:00 最近运行结果为 0。注册表仍为 Updated / AvailableUpdates=0；未读到对应 Error 值名。
- 实际 Boot Manager 资源 SVN 为 11.0，本地待应用 DBXUpdateSVN 载荷也为 11.0；已读取的固件 dbx 未检出对应 Boot Manager SVN 记录。
- 最近现存 TPM-WMI 1796 仍为 2026-08-15，错误 0x80070057；未建立其与当前故障的因果关系。
- WinRE Enabled，位于 Partition 5，WIM 版本 `26100.9444`，内部实际引用的 winload.efi 可提取、验证。另有 OEM `RecoveryImage\install.wim`，版本 `26100.2314`，以及 COLORFUL 一键还原的菜单配置；尚未实启动或执行恢复。
- BIOS/主板序列及产品识别编号字段中检到了 `V360…`，完整序列只保存在仓库外；尚未确认具体 Clevo 平台/固件适用性。
- ESP 100 MiB 原始卷镜像的两次源读取和目标 SHA-256 一致；最终原样文件备份 149/149 校验通过。原始数据位置与早期工作副本的限制见最新报告。
- 当前 Secure Boot 关闭；本阶段没有开启、重启或执行修复。静态结果不能证明开启后固件放行，也没有新确认历史故障仍复现。

完整来源、日期和结论边界见 [2026-10-04 管理员调查](INVESTIGATION-2026-10-04.md)。以下为此前 BIOS 现场观察，保留其历史性质。

## 固件/硬件

- Insyde H2O BIOS。
- BIOS Version: `1.07.05COL03`
- KBC/EC Version: `1.09.04CF1`
- ME FW Version: `16.1.32.2473`
- CPU: Intel Core i9-13900HX
- Memory Size: 16384 MB
- DRAM Frequency: 5600 MHz

> 对话里曾提到“14900HX + RTX 5060”的新机设想/对比，但本次 BIOS 截图实际显示的是 **i9-13900HX**。不要把两者混写。

## TPM

Security -> TPM Configuration 页面：

- `TPM2.0 Device Found`
- `Clear TPM = Disabled`

本次调查没有执行 Clear TPM。

## Secure Boot 初始状态

Administer Secure Boot 页面：

- `Secure Boot Database = Installed and Locked`
- `Secure Boot Status = Disabled`
- `User Customized Security = NO`
- `Enforce Secure Boot = Disabled`
- `Erase all Secure Boot Settings = Disabled`
- `Restore Secure Boot to Factory Settings = Disabled`
- 存在 `PK Options`

## 关键 A/B 测试

### Enforce Secure Boot = Disabled

Windows 可以正常启动。

### Enforce Secure Boot = Enabled

保存并重启后出现：

`Windows Boot Manager boot failed.`

随后出现：

`Default Boot Device Missing or Boot Failed.
Insert Recovery Media and Hit any key
Then Select 'Boot Manager' to choose a new Boot Device or to Boot Recovery Media`

因此可以确认：

- 固件能看到/尝试 Windows Boot Manager；
- 一旦 Secure Boot enforcement 生效，Windows 启动路径被拒绝；
- 关闭 enforcement 后系统可恢复启动。

这使“普通 SSD 完全不识别”或“根本没有 Windows Boot Manager”变得不符合现象。

## Secure Boot 菜单的启动周期限制

失败启动后再进入管理页，会显示：

`The operation is only allowed before booting any boot device!!! Please reset system and enter this menu directly if need use this operation.`

彻底重启并在尝试任何 boot device 之前直接进入 BIOS/Secure Boot 菜单，可以再次修改。

## EFI 文件浏览器

在：

`Administer Secure Boot -> Select a UEFI file as trusted for execution`

可见：

- `bootmgfw.efi`
- `bootmgr.efi`
- `memtest.efi`
- `SecureBootRecovery.efi`
- 多个语言资源目录，例如 `zh-CN`、`zh-TW`

曾尝试围绕 `bootmgfw.efi` 做“trusted for execution”操作，但用户报告问题没有因此解决。由于当时每一步确认画面没有完整记录，暂不把“具体写入了什么 DB 项”视为已确认。

## CSM / Legacy

在已经浏览和拍摄的菜单中未发现 CSM/Legacy 开关。

当前现象也显示固件正在以 UEFI 方式调用 Windows Boot Manager，但“没有看到 CSM”不等于证明固件代码里绝对没有兼容模块。
