# 已确认事实

更新日期：2026-10-03

本文件只放亲自观察到或截图能够直接支持的事实；推测放到其他文档。

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
