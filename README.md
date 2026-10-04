# laptop-secure-boot-investigation

记录并处理一台 COLORFUL 笔记本的 Secure Boot 启动失败问题。目标是把“亲自观察到的事实”“当前假设”“下一步恢复方案”分开，避免边试边忘，也方便交给 Codex 继续做 U 盘工具。

## 当前结论

**2026-10-04 管理员取证已执行：唯一 ESP、固件实际启动项与 BCD 路径一致；PK/KEK/db/dbx 全部完整解析，实际 Boot Manager 的 2023 签名、PE 摘要及到 db 的证书签名链验证通过，所检查文件未命中当前 dbx。根因仍未确定；没有证据支持直接用 BCDBoot 重建启动环境。**

`SecureBoot=0`、`SetupMode=0`；Secure-Boot-Update 任务存在、已启用，今天最近运行返回 0。已在仓库外完成 ESP 的 100 MiB 原始镜像及 149/149 文件备份校验。实际 Boot Manager SVN 与本地待应用载荷均为 11.0，当前 dbx 未检出对应 SVN 条目。Windows 侧检查不能代替开启 Secure Boot 后的固件实测。

CLI 继续排查时先读 [最新调查报告](docs/INVESTIGATION-2026-10-04.md)、[AGENTS.md](AGENTS.md) 和 [CLI-HANDOFF.txt](CLI-HANDOFF.txt)。下面保留此前 BIOS 观察和调查背景。

2026-10-04 临时任务见 [TEMP-TASK.md](TEMP-TASK.md)：**本轮八项检查与条件判断已完成**，任务文件已逐项勾选，列出备份位置和后续待确认事项，原任务正文保留供追溯。详细证据、读取错误和未实测事项见[报告的第二阶段](docs/INVESTIGATION-2026-10-04.md#第二阶段执行-temp-taskmd-的八项任务)。按任务条件，本轮未生成修复脚本、未执行修复或重启。WinRE 可读取，另发现 OEM 恢复映像与“一键还原”配置；尚未验证实际恢复行为。

历史 A/B 测试将问题指向 **启用 UEFI Secure Boot 后的 Windows EFI 启动链验证过程**；当前取证未发现实际启动项目标错误或所检查 EFI 映像的 dbx 哈希撤销。历史故障是否仍可复现尚未得到本轮确认。

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

本节列出此前提出的假设；2026-10-04 的证据已降低“单纯缺少 Windows UEFI CA 2023”以及旧启动文件尚未迁移的优先级。实际启动文件为 2023 签名。其他撤销规则、固件模式与密钥配置、后续启动链及 OEM 固件兼容性仍待核查。

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

**当前下一步**：确认历史开启 enforcement 的故障是否仍可复现、期间是否改过设置，再结合原始启动项、证书指纹和 `V360…` 身份线索核实 OEM 固件适用性。变量、任务、数据库和本轮相关 EFI 文件的静态检查已完成；任何开启 Secure Boot/重启或具体修复仍须按项目边界另行明确授权。详见[最新调查报告](docs/INVESTIGATION-2026-10-04.md)。

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
