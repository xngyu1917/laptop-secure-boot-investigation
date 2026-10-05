# laptop-secure-boot-investigation

记录并处理一台 COLORFUL 笔记本的 Secure Boot 启动失败问题。目标是把“亲自观察到的事实”“当前假设”“下一步恢复方案”分开，避免边试边忘，也方便交给 Codex 继续做 U 盘工具。

## 当前结论

**2026-10-05 现场复测进一步收窄范围：Windows 已重新安装但故障不变；Enforce Secure Boot 关闭时内部 Windows 与 Windows 安装 EFI USB 均可启动，开启并保存重启后，内部 Windows Boot Manager 与独立 EFI USB 均直接 `boot failed`。结合 2026-10-04 的管理员静态取证，当前更应优先怀疑固件执行 Secure Boot 验证时的兼容性/状态问题，而不是单一 Windows、BCD、ESP 或 `bootmgfw.efi` 故障。**

此前管理员取证已确认：唯一 ESP、固件实际启动项与 BCD 路径一致；PK/KEK/db/dbx 全部完整解析，实际 Boot Manager 的 Windows UEFI CA 2023 签名、PE 摘要及到 db 的证书签名链验证通过，所检查文件未命中当前 dbx；Boot Manager SVN 也已检查。BIOS 现场 DB 页面同时直接显示 Windows UEFI CA 2023、Microsoft UEFI CA 2023 等条目。因此“单纯缺少 2023 CA”已明显降低优先级。

CLI 继续排查时先读 [最新现场复测](docs/INVESTIGATION-2026-10-05.md)、[2026-10-04 管理员调查](docs/INVESTIGATION-2026-10-04.md)、[AGENTS.md](AGENTS.md) 和 [CLI-HANDOFF.txt](CLI-HANDOFF.txt)。

2026-10-04 临时任务见 [TEMP-TASK.md](TEMP-TASK.md)：**本轮八项检查与条件判断已完成**，任务文件已逐项勾选，列出备份位置和后续待确认事项，原任务正文保留供追溯。详细证据、读取错误和未实测事项见[报告的第二阶段](docs/INVESTIGATION-2026-10-04.md#第二阶段执行-temp-taskmd-的八项任务)。按任务条件，本轮未生成修复脚本、未执行修复或重启。WinRE 可读取，另发现 OEM 恢复映像与“一键还原”配置；尚未验证实际恢复行为。

2026-10-05 已重新确认历史故障仍可复现，并增加独立 EFI USB 对照：**Secure Boot OFF 时 Windows/USB 均可启动；Secure Boot ON 时 Windows/USB 均 boot failed。** 这把范围进一步从“Windows EFI 启动链”收窄到固件 Secure Boot enforcement 的共同验证层。

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
- 2026-10-05 使用独立 EFI USB 复测：
  - `Enforce Secure Boot = Disabled`：USB 可启动；
  - `Enforce Secure Boot = Enabled`：固件显示 `EFI USB Device (VendorCoProductCode) boot failed.`。
- BIOS `DB Options` 现场显示 7 个 PKCS7 条目，其中包括 `Microsoft Windows Production PCA 2011`、`Windows UEFI CA 2023`、`Microsoft UEFI CA 2023`、`Microsoft Option ROM UEFI CA 2023`；`DBX Options` 显示大量 SHA256 撤销项。
- `Select a UEFI file as trusted for execution` 明确说明会把指定 EFI image hash 加入 allowed database；本轮没有执行该写入。
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

本节列出此前提出的假设；2026-10-04 的证据已降低“单纯缺少 Windows UEFI CA 2023”以及旧启动文件尚未迁移的优先级。实际启动文件为 2023 签名。其他撤销规则、固件模式与密钥配置、后续启动链及 OEM 固件兼容性仍待核查。

还没有最终证明根因。当前优先级调整为：

1. OEM/Insyde 固件实际执行 Secure Boot 验证时的兼容性或运行时状态问题；
2. 当前 PK/KEK/db/dbx/相关安全变量的组合需要 OEM 认可的恢复或重新初始化；
3. 2011 → 2023 Secure Boot 迁移与该 BIOS/EC 版本之间存在 OEM 兼容问题。

单纯“内部 Windows Boot Manager 不受信任”已降低优先级，因为独立 EFI USB 在 enforcement 开启时也被同一固件拒绝。

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

**当前下一步**：联系 COLORFUL/OEM，携带 BIOS `1.07.05COLO3`、KBC/EC `1.09.04CF1`、ME `16.1.32.2473` 与 Windows/USB 的 Secure Boot ON/OFF A/B 结果，确认是否存在准确匹配本机的新版 BIOS/EC、Secure Boot/2023 证书兼容修复，以及 OEM 是否建议执行 `Restore Secure Boot to Factory Settings`。在得到 OEM 确认前，不执行 Erase Keys、删除 DBX、手工 trust EFI 或交叉刷其他机型固件。详见[最新现场复测](docs/INVESTIGATION-2026-10-05.md)。

以下 U 盘步骤保留为条件性恢复方案；目前尚未确认本机适用，不应直接据此执行：

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
