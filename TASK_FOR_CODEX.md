# Codex 任务：制作安全的 Secure Boot Recovery U 盘工具

请先完整阅读 README.md、docs/CONFIRMED.md、docs/RECOVERY-PLAN.md。

## 任务

实现一个 Windows PowerShell 工具：

`scripts/make-secureboot-recovery-usb.ps1`

用于把本机的：

`C:\Windows\Boot\EFI\SecureBootRecovery.efi`

安全复制成指定 FAT32 U 盘的：

`X:\EFI\BOOT\bootx64.efi`

## 强制安全约束

1. **禁止自动格式化磁盘。**
2. **禁止自动选择“第一个可移动磁盘”。**
3. 必须由用户显式传入目标盘符，例如：
   `-DriveLetter E`
4. 拒绝 `C:` 和 Windows 系统盘。
5. 检查目标卷文件系统必须是 FAT32；否则停止并给出人工格式化说明。
6. 检查目标盘尽量应为 removable；如果 Windows 报告类型不明确，必须二次确认，不能静默继续。
7. 写入前显示：
   - 源文件
   - 目标盘
   - 目标路径
   - 卷标/容量/文件系统
8. 默认要求用户输入明确确认词，例如 `YES`。
9. 创建 `EFI\BOOT` 后复制文件。
10. 复制后计算并比较源/目标 SHA-256。
11. 输出下一步 BIOS 操作说明，但不要自动修改任何 BIOS/TPM/Secure Boot 配置。
12. 不下载未知二进制文件；优先使用 Windows 自带的 `SecureBootRecovery.efi`。
13. 如果源文件不存在，停止并说明需要先把主电脑 Windows 更新到微软支持该文件的版本；不要从非官方镜像自动下载。

## 附加文件

请同时生成：

- `scripts/verify-secureboot-state.ps1`
  - 只读收集：
    - `Confirm-SecureBootUEFI`
    - `UEFICA2023Status`
    - `UEFICA2023Error`
    - `UEFICA2023ErrorEvent`
    - `AvailableUpdates`
    - Secure-Boot-Update 任务存在/启用/最近运行状态
  - 不能修改 registry / scheduled task / firmware。

- `docs/USB-HOWTO.md`
  - 给普通用户的一页中文步骤。
  - 明确写出“不要 Clear TPM / Erase keys”。

## 验收

- PowerShell 5.1 和 PowerShell 7 均尽量兼容。
- 脚本具有 `-WhatIf` 或等价 dry-run。
- 所有危险路径默认 fail closed。
- README 增加运行示例。
- 不做任何需要管理员之外更高特权的固件写入。
