# Secure Boot 现场记录：清除、恢复默认与手工信任测试

记录日期：2026-10-06。**故障未解决，根因未确定。本文不排列原因可能性。**

## 1. 来源、时间与本轮范围

来源是本次对话中用户上传的 BIOS 照片，以及操作后的即时文字反馈。整理者没有远程控制电脑，没有在本轮运行本机命令、读取新数据库或查看新的启动日志。照片原件尚未上传本仓库；下面以画面内容与 BIOS 显示时间索引，不能把文字记录当作原始二进制证据。

本轮照片的 BIOS 日期显示为 2026/10/07，可见时间大约从 01:25 到 02:02。文件名按本次对话日期 2026-10-06 记录。BIOS 时钟和时区没有核实，不能据此推断时钟异常与故障有关；以下以对话先后顺序为准。

设备沿用历史身份：COLORFUL P16 Pro、Insyde H2O、i9-13900HX、16 GB。历史 SMBIOS 版本原文为 `1.07.05COLO3`，KBC/EC `1.09.04CF1`，ME `16.1.32.2473`；旧转录存在 O/0 差异。准确子型号与适用固件包未在本轮核实。

本文用三类来源标签：

- **[照片]**：能够从本轮照片直接看到的菜单、条目或错误文字。
- **[用户反馈]**：用户明确说已做或给出的结果，但没有覆盖全部过程的照片／日志。
- **[历史报告]**：仓库既有记录，只对其所注明的时间和对象成立。

## 2. 本轮开始前的历史记录

[2026-10-05 报告](INVESTIGATION-2026-10-05.md)记录：用户已重新安装过 Windows，开启安全启动后问题仍然出现；关闭时内部 Windows 和该 Windows 安装 USB 可启动，开启时两者分别出现启动失败。该 USB 的实际 EFI 入口、签名版本和制作方式未完整固定，不能当成已完成 2011／2023 签名配对测试。

[2026-10-04 管理员报告](INVESTIGATION-2026-10-04.md)记录过当时的 ESP、启动项、文件签名／摘要、数据库和 dbx 比对。本轮没有重新执行这些检查。用户所述重装与这些历史测量之间的准确时间关系、后续文件是否变化，不能仅凭对话补齐；不把旧结果写成现在仍完全相同。

## 3. 阶段 A：清除安全启动设置

### A1. 清除前的菜单与选择

**[照片，约 01:25～01:28]** `Administer Secure Boot` 页面可见：

```text
Enforce Secure Boot                         Disabled
Erase all Secure Boot Settings              Disabled -> Enabled
Restore Secure Boot to Factory Settings     Disabled
```

选中清除项时，固件帮助文字说明会清除四个变量：`PK`、`KEK`、`db`、`dbx`。用户确认选择后，照片显示 Erase=Enabled，其他两行仍为 Disabled；随后用户按保存重启流程返回同一菜单。

### A2. 返回 BIOS 后的可见状态

**[照片，01:29:18]**

```text
Secure Boot Database          Unlocked
Secure Boot Status            Disabled
User Customized Security      YES
Enforce Secure Boot           Disabled（灰色，不可选）
Erase all Secure Boot Settings Disabled
Restore Secure Boot to Factory Settings Disabled
```

这是清除操作前后的实际界面变化。**没有在这个阶段导出 PK/KEK/db/dbx，或拍下每个数据库为空的完整页面；没有读取 SetupMode 的数值。** 因此只记录清除操作和上述状态，不写成“已逐字节证明四库为空”“整个 BIOS 已重装”或“所有固件状态已初始化”。

菜单清除说明只列出四个变量；未检查 DBT、DBR 或其他状态是否变化。

## 4. 阶段 B：恢复默认配置并测试

**[用户反馈]** 用户按 `Restore Secure Boot to Factory Settings` 的操作流程保存、重启并直接进入 BIOS，报告安全启动开关已经自动开启，自己没有另外手动开启，然后尝试启动系统。恢复操作的全部确认画面和恢复后主状态页面未完整留图。

**[照片]** 随后的错误为：

```text
Windows Boot Manager boot failed.
```

接着出现：

```text
Default Boot Device Missing or Boot Failed.
Insert Recovery Media and Hit any key
Then Select 'Boot Manager' to choose a new Boot Device or to Boot Recovery Media
```

**[照片]** 本轮尝试启动后再进入相关安全设置，还出现：

```text
The operation is only allowed before booting any boot device!!!
Please reset system and enter this menu directly if need use this operation.
```

这是该次菜单操作的限制提示，不把它另列为已确定的启动失败原因。

**[用户反馈]** 用户随后说已关闭安全启动，进入 DB Options 查列表。

### B1. 恢复后、手工添加前的 DB 页面

**[照片，01:39:55]** 页面标题为 `Administer Secure Boot > DB Options`，可见 7 条，类型均由界面标为 `[PKCS7]`：

1. Microsoft Windows Production PCA 2011
2. Windows UEFI CA 2023
3. Microsoft Corporation UEFI CA 2011
4. Microsoft UEFI CA 2023
5. Microsoft Option ROM UEFI CA 2023
6. Secure Certificate
7. Cus CA

这些名称与此前照片中的名称一致。因此不能将“恢复后页面没有 Windows UEFI CA 2023”记为事实。

但本轮没有比较证书 DER 指纹、数据库原始字节或当前 dbx，也没有重新验证当前实际启动文件。**同名条目存在，不等于恢复前后所有数据相同或已经证明能够启动。**

## 5. 阶段 C：手工信任文件，先测试一次

### C1. 文件浏览过程

**[照片，01:50～01:56]** 用户进入 `Select a UEFI file as trusted for execution`。卷列表可见一个 `NO VOLUME LABEL` 和两个标为 `NTFS` 的入口；截断的设备路径不足以独立确定完整分区身份。

用户先进入一个只显示 `bootx64.efi` 的文件目录，点击文件后出现添加确认框。按对话导航返回上一级的照片可见并列的 `<Boot>` 和 `<Microsoft>`；随后进入 Microsoft 下的 Boot，照片显示并选中了 `bootmgfw.efi`，同页还可见 `bootmgr.efi`、`memtest.efi`、`SecureBootRecovery.efi` 以及语言／资源目录。

这段浏览对应的两个路径是：

```text
EFI\Boot\bootx64.efi
EFI\Microsoft\Boot\bootmgfw.efi
```

本轮是按文件浏览器导航定位；没有重新读取固件 Boot####、GPT 分区标识、BCD 或实际启动文件身份，不将历史 Boot0001 映射当作本轮新检查。

### C2. 添加与第一次失败

**[照片／用户反馈]** 添加对话框显示：

```text
Add this hash image to allowed database (db)
```

用户在找到 `bootmgfw.efi` 后报告点击 Yes，窗口消失；用户表示已经打开安全启动，之后报告“失败了。没什么用”。这一轮没有新的错误画面或返回码，记录为**用户报告启动失败**，不补写为新拍摄的特定错误。

**[照片，01:59:57]** 再次进入 DB Options 后，前 7 个证书名称仍可见，新增第 08 条 `[SHA256]`，可辨识前缀为：

```text
F2 35 CD 21 C9 19 E0 5D A1 21 8E 8D 13 42 4F ...
```

这支持“添加操作后界面出现并在再次进入 BIOS 后仍能看到一个 SHA256 登记项”。该项与刚才操作存在时间上的对应；**本轮未导出完整值并独立复算当前文件的相应摘要，不能把前缀当成完整匹配证明。**

历史 10-04 报告包含同前缀的 PE/COFF 摘要，并明确区分它和整文件 SHA256；本轮没有据此直接补全截图中省略的值。

## 6. 阶段 D：重复项提示、追加更多条目、再次失败

### D1. 重复项提示

**[照片，02:00:47]** 后续一次选文件并确认 Yes 后，固件显示：

```text
[Error] There is already an identical signature in signature list.
```

照片本身没有包含当次选中的完整文件路径。**[用户反馈补充]** 用户随后明确补充：这里比较的是 `EFI\\Boot\\bootx64.efi` 与先前已加入的 `EFI\\Microsoft\\Boot\\bootmgfw.efi`，两者哈希值相同，因此再次添加时出现已有相同签名的提示。2026-10-04 的历史管理员报告也曾记录，当时实际 `bootmgfw.efi` 与备用 `bootx64.efi` 的整文件 SHA-256 相同。

本轮没有重新导出两个文件的完整哈希或确认 BIOS 此处采用的摘要口径，因此这里保留为“用户本轮确认哈希相同 + 历史报告曾记录整文件相同”，不把它扩展成其他文件也相同，也不等于固件启动时已执行放行。

### D2. 追加更多文件

**[用户反馈补充]** 用户说明这是一次“死马当活马医”的扩大白名单测试。除先前的 `bootmgfw.efi` 外，用户还对 `EFI\\Microsoft\\Boot` 中的 `bootmgr.efi`、`memtest.efi`、`SecureBootRecovery.efi` 执行了信任添加，并对 `zh-CN` 目录下可见的所有 `.mui` 文件执行了同类操作；`EFI\\Boot\\bootx64.efi` 也被尝试添加，但因与已有条目哈希相同出现重复提示。用户没有逐次保留每个文件的完整哈希和确认画面。

**[照片，02:02:23]** DB Options 列出 01～14：前 7 个证书名称，以及 08～14 共 **7 条可见 `[SHA256]` 登记项**。

这些 7 条可见登记与上述批量操作发生在同一阶段，但不能仅凭截图把每一条 SHA256 与某一个具体文件逐项绑定。尤其 `zh-CN` 下多个 `.mui` 的逐一结果没有单独记录；截图中的摘要也被截断。本文只记录用户明确说明的操作范围，不把它写成“整个启动链全部受信任”。

### D3. 最后一次结果

**[用户反馈]** 用户按再次开启安全启动并保存测试的流程继续，之后回复：**“失败”**。

没有提供这一次新的报错照片、返回码或启动日志。记录为追加更多条目后用户报告仍启动失败，不将它写成已独立验证的特定签名错误。

## 7. 最后状态与没有完成的事项

最后一张 DB 照片在末次失败前，显示 7 个证书名称与 7 条 SHA256；最后一次明确结果是用户报告失败。随后助手建议关闭安全启动并清理手工条目，但**建议不等于已执行**。截至用户要求更新 GitHub，没有末次之后开关状态、Windows 成功进入或再次清理／恢复的确认。

本轮没有以下新的执行证据：

- 恢复前后 PK/KEK/db/dbx 的完整导出、字节比较及证书指纹比较。
- 追加后所有 SHA256 与当前实际启动文件的逐项对应；当前 Boot####、分区和文件签名的重新采集。
- 清除阶段 SetupMode 的读取，或所有安全变量均已复位的检查。
- 对签名已明确核实的 Win10 安装 U 盘进行启动对照；只讨论过方案，没有制作、签名确认或结果反馈。
- 本轮恢复／追加之后对原安装 USB 重新完成一组开关对照。
- 运行 SecureBootRecovery.efi、再次安装 Windows、刷写 BIOS/EC、清 TPM，或手工删除 dbx 的新操作。清除选项帮助中包含 dbx，不应与另外手工编辑 dbx 混写。
- 设备加密状态的命令输出。用户表示没有设备加密，但本轮没有独立核实；不写成检查通过。

## 8. 交接时不可越过的证据边界

**本轮结论只到这里：清除／恢复默认后未解决，单次手工信任后未解决，追加更多登记项后用户仍报告失败。根因没有确定。**

以下是对本次聊天中部分过强表述的纠正，不是新的原因假设：

| 不沿用的表述 | 记录应写成 |
|---|---|
| “清空以后所有状态都干净了” | 清除后看到了具体界面变化；原始变量未逐项核实 |
| “恢复出厂还是失败，证明 BIOS 有 bug” | 此项操作未解决问题；未唯一确定根因 |
| “白名单已经写入，所以最前面的启动器已经放行了” | DB 中可见新增登记；没有执行跟踪证明放行 |
| “boot failed 就是签名验证失败” | 画面报告启动失败；具体返回状态与发生阶段未知 |
| “重复签名说明两个文件完全相同” | 固件报告登记项重复；完整路径和摘要规则仍须核对 |
| “增加 7 个哈希等于全部启动文件都信任了” | 可见 7 条 SHA256；完整文件映射与覆盖范围未知 |
| “旧检查全通过，所以当前启动链和撤销情况已经排除” | 旧报告对当时采集对象有效；本轮变化后没有重测 |

下一位接手者应先确认最新机器状态和证据缺口，再提出可区分结果的检查。不要因上一位 AI 的语气、多个 AI 的一致意见或操作没有成功，就替某一根因定案。本次写入仅为文档记录，不授权新的电脑操作。
