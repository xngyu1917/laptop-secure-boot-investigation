# 已观察事实索引

更新日期：2026-10-06。根因未确定。本文件区分截图、用户反馈和历史报告，不把上一位 AI 的解释写成已验证结果。最新完整过程见 [10-06 现场记录](INVESTIGATION-2026-10-06.md)。

## 本轮：截图直接可见

| 阶段 | 可直接核对的内容 |
|---|---|
| 清除选项 | Erase all Secure Boot Settings 被设为 Enabled；帮助说明列出 PK、KEK、db、dbx；当时 Enforce 与 Restore 均为 Disabled |
| 清除后返回 | Database=Unlocked；Status=Disabled；User Customized Security=YES；Enforce 灰色；Erase 回到 Disabled |
| 恢复后第一次测试 | 照片显示 Windows Boot Manager boot failed，随后默认启动设备失败；再次访问安全设置时出现“本轮启动过设备，需重启直接进入菜单”的提示 |
| 恢复后、手工添加前的 DB | 页面显示 7 个 PKCS7 名称，其中有 Windows UEFI CA 2023，完整名称见下文 |
| 文件浏览 | 可见 EFI 下 Boot 和 Microsoft 并列；EFI\Boot 内有 bootx64.efi；沿 Microsoft\Boot 进入后照片选中 bootmgfw.efi |
| 单次信任后回看 DB | 出现第08条 SHA256，前缀 F2 35 CD 21 C9 19 E0 5D A1 21 8E 8D 13 42 4F ... |
| 再次添加 | 出现 There is already an identical signature in signature list；该次完整选中文件路径未显示 |
| 追加更多条目后 | 最后一张 DB 照片列出前7条证书名称及08～14共7条可见 SHA256；不是已逐项核实名称的7个文件清单 |

本轮 DB 页面显示的 7 个证书名称：

1. Microsoft Windows Production PCA 2011
2. Windows UEFI CA 2023
3. Microsoft Corporation UEFI CA 2011
4. Microsoft UEFI CA 2023
5. Microsoft Option ROM UEFI CA 2023
6. Secure Certificate
7. Cus CA

这是界面名称记录；没有本轮新导出的证书内容、指纹或完整 dbx 数据。

## 本轮：用户明确反馈，未全部留图

用户报告按清除／恢复流程操作并重启；恢复后安全启动已自动开启。首次失败后说已关闭安全启动。之后报告对 `bootmgfw.efi` 的添加确认框点击 Yes、开启安全启动测试仍失败。

用户随后补充：`EFI\\Boot\\bootx64.efi` 与 `EFI\\Microsoft\\Boot\\bootmgfw.efi` 的哈希值相同；历史 10-04 管理员报告也曾记录这两个文件当时整文件 SHA-256 相同。之后用户为了扩大白名单测试，还信任了 `bootmgr.efi`、`memtest.efi`、`SecureBootRecovery.efi`，并对 `zh-CN` 下可见的全部 `.mui` 文件执行同类操作；`bootx64.efi` 再添加时因重复哈希未形成独立新条目。最终再次测试仍由用户报告“失败”。

首次恢复后的失败有照片。单次手工信任及最终多条目后的失败是文字反馈，没有各自的新报错照片／日志。不要把助手反复复述的同一句报错当作多份独立证据，也不要把可见 7 条 SHA256 机械对应成 7 个已独立核实的文件。

最后失败之后是否已关闭安全启动、是否能进 Windows、是否清理新增条目，尚未收到确认。助手给出的收尾建议不是操作记录。

## 本轮没有重新确认

没有逐项证明清除后的四库为空；没有实测 SetupMode；没有恢复前后原始字节比较；没有当前完整启动项／分区身份复核；没有新增哈希与当前文件的独立完整摘要比对；没有重新检查当前 dbx、所有策略或所有启动环节。

没有本轮已核实签名的 Win10 安装 U 盘对照结果。没有运行恢复工具、再次重装、刷 BIOS/EC 或清 TPM 的新记录。用户称没有设备加密，未提供本轮独立状态检查。

## 历史：2026-10-05 Windows／USB 对照

来源：[10-05 历史报告](INVESTIGATION-2026-10-05.md)。

- 用户报告已重装过 Windows，问题仍然存在。
- 关闭 Enforce 时内部 Windows 可启动；开启并保存重启后，Windows Boot Manager 启动失败。
- 同一安装 USB 关闭时可启动／安装，开启时出现 EFI USB Device (VendorCoProductCode) boot failed。
- 当时 DB 页面显示上述 7 个名称，DBX 页面可见多条 SHA256。
- 当日记录没有执行清除、恢复默认或有完整记录的手工添加。本轮已经进行了这些新操作，不能继续把它们列为“从未试过”。

旧 USB 的实际入口、制作方式和签名版本未完整固定；不把该对照扩大为所有 EFI 程序均不能启动。

## 历史：2026-10-04 管理员检查

来源：[10-04 管理员报告第二阶段](INVESTIGATION-2026-10-04.md#第二阶段执行-temp-taskmd-的八项任务)。下列值仅代表当时的测量，本轮没有重新取得相同输出。

| 项目 | 当日报告记录 |
|---|---|
| 管理员与状态 | 管理员 token=True；Confirm-SecureBootUEFI=False；SecureBoot=00；SetupMode=00 |
| 数据库 | PK/KEK/db/dbx 为 844/3066/9310/19996 字节；四库解析没有剩余字节；分别有1/2/7张 X.509 和416条 SHA256 |
| 启动路径 | 1块物理磁盘、5分区、1个 ESP；BootCurrent=0001，Boot0001、GPT/LBA、Disk0 Partition2、BCD 指向对应的 EFI\Microsoft\Boot\bootmgfw.efi |
| 文件验证 | 13个相关 EFI 路径的 CMS 签名与 PE 摘要检查通过；实际 Boot Manager 与 db 中 Windows UEFI CA 2023 的签名关系验证通过；所查对象未匹配当时 dbx |
| 文件一致性 | 实际 bootmgfw.efi、备用 bootx64.efi 与 Windows EFI_EX 副本整文件相同；普通 Windows EFI 副本整文件不同但 PE 摘要相同；不把这两种哈希混用 |
| SVN 与日志 | Boot Manager SVN=11.0；本地待应用载荷也是11.0；当时 dbx 未检出相应 SVN 记录；历史1796最近为2026-08-15，错误0x80070057，未建立启动故障因果关系 |
| 更新任务 | Secure-Boot-Update 存在且启用，最近结果0；注册表 Updated/AvailableUpdates=0；这些不等于固件启动已验证 |
| 恢复与备份 | WinRE 配置／文件和 OEM 恢复资源可读取，未实启动；100 MiB ESP 镜像及149/149原样文件备份校验通过，备份位于同一SSD，原始数据未上传 |

原始证据路径、各文件的完整哈希、早期工作副本被分析工具改写的限制及读取错误，保留在原报告与 [历史 CLI 交接](CLI-HANDOFF-2026-10-04.txt)。本轮未检查备份是否仍存在，不覆盖或重新解释旧输出。

## 设备及其他历史观察

历史身份为 COLORFUL P16 Pro、i9-13900HX、16GB，BIOS 原文 1.07.05COLO3、KBC/EC 1.09.04CF1、ME 16.1.32.2473；旧手抄有 O/0 差异。V360…只是在身份字段中的历史线索，不是已核实的子平台。TPM 页面曾显示 TPM2.0 Device Found 和 Clear TPM=Disabled。

此前浏览的 BIOS 菜单未发现 CSM/Legacy 选项，这只是检查范围内未见。目录页面差异只记录实际所在层级，不据此诊断重装造成文件异常。
