# 项目说明

本项目调查七彩虹 P16 Pro 开启 Secure Boot 后 Windows Boot Manager 启动失败的问题。面向用户使用中文说明。用户最新明确指令优先于项目文档。

## 从哪里继续

1. 先读 `docs/INVESTIGATION-2026-10-05.md`，这是最新现场复测；再读 `docs/INVESTIGATION-2026-10-04.md` 获取此前完整管理员静态取证。
2. 再读 `CLI-HANDOFF.txt`，其中保留本机路径、已运行命令和调查背景。
3. `docs/CONFIRMED.md` 包含历史 BIOS 观察与新确认事实。
4. `docs/RECOVERY-PLAN.md` 是有适用条件的历史恢复方案。实际 EFI 启动文件已由 Windows UEFI CA 2023 签发，固件 db 也检出了这张证书；2026-10-05 又确认独立 EFI USB 在 enforcement 开启时同样 boot failed，因此不要把“补 2023 CA”或单个 Windows EFI 文件当作首要修复。
5. `TASK_FOR_CODEX.md` 是此前拟议的 U 盘工具开发任务。只有用户请求制作工具时才实施；阅读它不代表授权运行恢复工具。
6. `TEMP-TASK.md` 的 2026-10-04 八项取证与条件判断已完成。2026-10-05 已重新实测 Secure Boot ON/OFF，并加入 USB 对照；不要再次把“确认故障是否仍可复现”当成待办。故障根因尚未唯一确定，当前优先转 OEM BIOS/EC 与 Secure Boot 数据库恢复适用性。

## 排查与记录

- 默认先进行只读诊断，分开记录亲自读取的事实、用户回传的结果、工作假设和未完成检查。
- 固件变量读取报错必须保留错误，不能把读取失败转换成 `False` 或“证书不存在”。
- ASCII 搜索证书名称是初筛，不代替完整 EFI_SIGNATURE_LIST、证书链和 dbx 哈希检查。
- `C:\Windows\Boot\EFI\bootmgfw.efi` 是 Windows 目录副本；应核对 EFI 系统分区实际启动文件及固件启动项。两者签发者不同不能混写。
- Windows Authenticode `Valid` 不等于固件一定允许启动；存在可信证书也不能单独证明固件正常。
- 历史事件日志须注明日期，不能把 2026 年 8 月的事件直接当成当前故障。
- CLI 需要读取 UEFI 变量时先检查执行进程是否为管理员。`--yolo` 不会提升 Windows 权限。
- 新证据写入带日期的 `docs/INVESTIGATION-YYYY-MM-DD.md`，同步更新 README 的当前结论，保留有价值的历史观察。

## 操作边界

未获得用户对具体修复操作的明确授权时，不清 TPM，不删除或重置 PK/KEK/db/dbx，不恢复出厂 Secure Boot 密钥，不刷 BIOS、不重装、不格式化磁盘、不运行 bcdboot 修复、不修改固件/注册表/计划任务、不自动重启。

如用户请求制作恢复 U 盘，遵守 `TASK_FOR_CODEX.md` 的目标盘显式选择、禁止自动格式化、FAT32 检查、写入前确认和 SHA-256 校验约束。

不要提交登录凭据、BitLocker 恢复密钥或未经审查的原始系统数据。当前仓库为私有仓库；保持其现有可见性。
