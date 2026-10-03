# 笔记本安全启动问题调查

用于整理这台笔记本的故障现象、已做排查、证据和后续讨论。

## 当前状态（2026-10-03）

- 用户报告笔记本无法开启安全相关功能，并表示已排除多项问题；具体是否指 BIOS/UEFI 中的 Secure Boot，以及开启后的表现，仍待确认。
- 用户与 GPT 怀疑问题与微软 2011 → 2023 安全启动证书迁移有关。该判断目前是待验证假设，不是已确认根因。
- 尚未收到笔记本品牌型号、BIOS/UEFI 版本、Windows 版本、报错原文或检测日志。
- 仓库当前仅整理资料，尚未实施设备上的修复。

## 已核实的官方资料

微软正在将 2011 年签发的安全启动证书迁移到 2023 年证书：

| 2011 证书 | 到期日期 |
| --- | --- |
| Microsoft Corporation KEK CA 2011 | 2026-06-24 |
| Microsoft UEFI CA 2011 | 2026-06-27 |
| Microsoft Windows Production PCA 2011 | 2026-10-19 |

微软明确说明，旧证书到期本身不会使已有 Windows 系统突然无法正常启动。需要区分证书到期、信任数据库缺少证书、引导程序签名不匹配、撤销以及固件更新失败。

官方排障指南记录了两类相关情形：恢复旧版固件默认密钥后，缺少已更新引导程序需要的 2023 证书；固件更新证书时错误覆盖安全启动数据库。是否适用于本机，需要设备证据支持。

参考：

- [微软安全启动证书到期与更新说明](https://support.microsoft.com/en-gb/servicing/os/secure-boot/2025/06/windows-secure-boot-certificate-expiration-and-ca-updates)
- [微软安全启动排障指南](https://support.microsoft.com/en-us/servicing/os/secure-boot/2026/03/secure-boot-troubleshooting-guide)
- [微软官方 Secure Boot Objects 仓库](https://github.com/microsoft/secureboot_objects)

## 待补充材料

- 笔记本品牌、完整型号、BIOS/UEFI 版本与日期、Windows 版本。
- 具体故障：选项灰色、设置保存后失效、启用后无法启动、签名报错，或其他表现。
- 已做排查的操作与结果，尤其是更新固件、恢复默认密钥和修复引导的时间。
- GPT 之前的分析、检测输出和依据；明确区分观察结果与推断。

上传资料前移除密码、私钥、BitLocker 恢复密钥等敏感内容。
