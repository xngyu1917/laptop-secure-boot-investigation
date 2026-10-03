# laptop-secure-boot-investigation

记录并处理一台 COLORFUL 笔记本的 Secure Boot 启动失败问题。目标是把“亲自观察到的事实”“当前假设”“下一步恢复方案”分开，避免边试边忘，也方便交给 Codex 继续做 U 盘工具。

## 当前结论

问题已经明显缩小到 **UEFI Secure Boot 对 Windows EFI 启动链的信任/签名验证层**，而不是普通的“硬盘不存在”或“Windows 完全不是 UEFI 安装”。

已经亲自验证：

- BIOS 为 Insyde H2O 图形界面。
- BIOS 版本：`1.07.05COL03`。
- CPU：Intel Core i9-13900HX。
- 内存：16 GB，BIOS 显示 DRAM 5600 MHz。
- KBC/EC：`1.09.04CF1`。
- ME FW：`16.1.32.2473`。
- TPM 页面显示 `TPM2.0 Device Found`，`Clear TPM = Disabled`。
- Secure Boot 页面初始状态：
  - `Secure Boot Database = Installed and Locked`
  - `Secure Boot Status = Disabled`
  - `User Customized Security = NO`
  - `Enforce Secure Boot = Disabled`
  - `Erase all Secure Boot Settings = Disabled`
  - `Restore Secure Boot to Factory Settings = Disabled`
- 将 `Enforce Secure Boot` 改为 Enabled 后，固件立即尝试启动 Windows Boot Manager，但出现：
  - `Windows Boot Manager boot failed.`
  - 随后 `Default Boot Device Missing or Boot Failed.`
- 把 `Enforce Secure Boot` 改回 Disabled 后，Windows 可以恢复正常启动。
- 在 `Select a UEFI file as trusted for execution` 文件浏览器中能看到：
  - `bootmgfw.efi`
  - `bootmgr.efi`
  - `memtest.efi`
  - `SecureBootRecovery.efi`
- 失败启动后，如果已经尝试过某个 boot device，本轮启动中再进入 Secure Boot 管理会提示：
  - `The operation is only allowed before booting any boot device!!!`
  - 需要彻底重启，并在任何启动设备被尝试前直接进入 BIOS/Secure Boot 管理。
- 在已检查到的 BIOS 菜单中没有发现 CSM/Legacy 选项；当前证据更符合纯 UEFI 启动路径。这里只记录“未发现”，不把它当作绝对证明。

## 当前工作假设

还没有最终证明根因。优先考虑：

1. Windows Boot Manager 的签名链与固件当前 Secure Boot DB 不匹配；
2. Windows 已经处于 2011 → 2023 Secure Boot 证书迁移的一部分状态，但固件 DB/KEK 未正确同步；
3. OEM/Insyde 固件对 DB 更新、追加或恢复默认值的行为存在兼容问题。

**不要把“2011 → 2023 迁移”写成已确认根因。**

微软 2026 Secure Boot 排障指南明确说明：
- Secure Boot 证书服务依赖 Windows 与 UEFI 固件共同工作；
- 某些固件会错误覆盖 DB 而不是追加；
- Secure Boot 重置后，如果默认 DB 不包含 Windows UEFI CA 2023，可能导致当前 Windows Boot Manager 被拒绝；
- `SecureBootRecovery.efi` 可从 FAT32 U 盘启动，用于把 Windows UEFI CA 2023 加回 DB。

官方资料：
- Microsoft Secure Boot troubleshooting guide:
  https://support.microsoft.com/en-us/servicing/os/secure-boot/2026/03/secure-boot-troubleshooting-guide
- 七彩虹官网“下载与服务”区域现已列出 Windows 证书更新说明：
  https://www.colorful.cn/

## 目前不要做

在原因未确认前，不要随意执行：

- `Clear TPM`
- `Erase all Secure Boot Settings`
- `Restore Secure Boot to Factory Settings`
- 手工删除 PK / KEK / db / dbx
- 直接格式化未知盘符的 U 盘

尤其不要清 TPM；设备若启用了 BitLocker/设备加密，可能触发恢复密钥要求。

## 下一步

1. 在另一台正常 Windows 电脑上确认：
   `C:\Windows\Boot\EFI\SecureBootRecovery.efi`
2. 制作 FAT32 U 盘：
   `\EFI\BOOT\bootx64.efi`
   其中 `bootx64.efi` 是复制并重命名后的 `SecureBootRecovery.efi`。
3. 受影响笔记本保持 `Enforce Secure Boot = Disabled`，从该 U 盘以 UEFI 方式启动恢复工具。
4. 工具运行后重新进入 BIOS，再测试 `Enforce Secure Boot = Enabled`。
5. 如果仍失败，回 Windows 收集 Secure Boot servicing 注册表、事件日志与 EFI 文件签名信息，再判断是固件 DB、KEK、Boot Manager 还是 OEM BIOS 问题。

详见：
- [docs/CONFIRMED.md](docs/CONFIRMED.md)
- [docs/RECOVERY-PLAN.md](docs/RECOVERY-PLAN.md)
- [TASK_FOR_CODEX.md](TASK_FOR_CODEX.md)
