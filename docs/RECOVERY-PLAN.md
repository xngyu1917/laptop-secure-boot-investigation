# 恢复方案（历史条件性备选，不是当前执行清单）

> 2026-10-06 状态提示：用户本轮已尝试清除、恢复默认和手工信任，仍报告失败；完整事实见 [10-06 现场记录](INVESTIGATION-2026-10-06.md)。本页下方的优先级和操作方案属于此前讨论，不代表已确认根因或最新授权。不能因阅读本页而重复初始化或自动制作／启动恢复工具。`SecureBootRecovery.efi` 在本轮没有运行记录；“恢复默认”也不是运行该工具。

## 2026-10-05 适用性再次下调（历史意见）

新的现场 A/B 已确认：`Enforce Secure Boot = Enabled` 时，不仅内部 Windows Boot Manager 启动失败，独立 EFI USB 也直接 `boot failed`；关闭 enforcement 后两者可启动。BIOS DB 页面同时直接显示 Windows UEFI CA 2023、Microsoft UEFI CA 2023 等证书。

因此，本页的 `SecureBootRecovery.efi` / “补 Windows UEFI CA 2023”路线进一步降低优先级。当前优先级改为：**先联系 COLORFUL/OEM，确认准确匹配本机的新版 BIOS/EC、Secure Boot/2023 证书兼容修复，以及是否应由 OEM 指导执行 Restore Secure Boot to Factory Settings。**

在 OEM 明确建议前，不执行 Erase Keys、删除 DBX、手工 trust 单个 EFI 文件或恢复 factory keys。

## 2026-10-04 适用性更新（历史意见）

固件 db 已检出 Windows UEFI CA 2023，实际 EFI 分区的 `bootmgfw.efi` 已由该 CA 签发，Windows Authenticode 检查为 `Valid`。因此，本页“追加 2023 证书”的方案目前应降低优先级，不是已经确认适用于本机的修复。

先按 [2026-10-04 管理员调查](INVESTIGATION-2026-10-04.md) 完成只读诊断。以下保留微软恢复工具的历史方案；只有进一步证据确认适用、且用户授权具体恢复操作后再执行。

## 历史方案目标

优先采用可逆、最小破坏的方式恢复 Secure Boot 对 Windows Boot Manager 的信任，而不是直接清空 TPM 或整套 Secure Boot keys。

## 为什么考虑 SecureBootRecovery.efi

微软 2026 Secure Boot 故障排除指南列出 Secure Boot recovery utility：

1. 在第二台、安装了 2024-07 或更高 Windows 更新的 Windows PC 上，从
   `C:\Windows\Boot\EFI\SecureBootRecovery.efi`
   获取恢复工具；
2. 放到 FAT32 U 盘：
   `\EFI\BOOT\bootx64.efi`
3. 从受影响设备启动该 U 盘；
4. 恢复工具会把 **Windows UEFI CA 2023** 加入 Secure Boot DB。

历史方案引用的官方资料（执行前须重新核对适用条件）：
https://support.microsoft.com/en-us/servicing/os/secure-boot/2026/03/secure-boot-troubleshooting-guide

## 制作 U 盘的安全原则

Codex 工具必须遵守：

- 默认 **绝不自动格式化** 任何磁盘。
- 用户必须明确指定目标盘符。
- 写入前检查：
  - 目标是可移动磁盘；
  - 文件系统为 FAT32；
  - 源文件存在；
  - 目标盘符不是系统盘；
- 复制前打印完整源路径和目标路径，并要求显式确认。
- 创建：
  `X:\EFI\BOOT\`
- 复制：
  `C:\Windows\Boot\EFI\SecureBootRecovery.efi`
  到：
  `X:\EFI\BOOT\bootx64.efi`
- 复制后验证目标文件存在、大小非 0，并比较 SHA-256。
- 不修改受影响笔记本的 TPM、PK、KEK、db、dbx。
- 不执行 `Erase all Secure Boot Settings`。
- 不执行 `Restore Secure Boot to Factory Settings`，除非后续证据明确要求且已确认 OEM factory DB 包含所需新证书。

## 受影响笔记本上的历史拟议流程

1. BIOS 保持 `Enforce Secure Boot = Disabled`。
2. 插入恢复 U 盘。
3. 从 Boot Manager 选择 UEFI U 盘启动。
4. 允许 `SecureBootRecovery.efi` 运行并重启。
5. 拔掉 U 盘。
6. 重新进 BIOS。
7. 将 `Enforce Secure Boot = Enabled`。
8. F10 保存并测试 Windows Boot Manager。

## 历史方案中的失败后检查

先关闭 Enforce，恢复 Windows 启动，再收集证据，不要继续“全开/全清”：

PowerShell / Registry：
- `UEFICA2023Status`
- `UEFICA2023Error`
- `UEFICA2023ErrorEvent`
- `AvailableUpdates`

任务计划：
- `\Microsoft\Windows\PI\Secure-Boot-Update`

事件日志：
- Secure Boot / servicing 相关事件

EFI：
- 当前 `bootmgfw.efi` 的 Authenticode 签名/证书链

OEM：
- 检查是否存在比 `1.07.05COL03` 更新的官方 BIOS。
