# Secure Boot 现场复测更新

日期：2026-10-05（America/La_Paz）。本页只记录 2026-10-04 管理员静态取证之后的新现场结果。此前完整静态调查见 [INVESTIGATION-2026-10-04.md](INVESTIGATION-2026-10-04.md)。

## 当前结论

根因仍未被唯一确定，但故障范围已经明显收窄。

此前 Windows 侧静态取证已确认：实际 ESP/BCD/固件 Boot#### 指向一致，实际 `bootmgfw.efi` 为 Windows UEFI CA 2023 签名，签名链可验证，固件 db 中存在对应 2023 CA，所检查 EFI 映像未命中当时解析出的 dbx SHA-256 撤销项，Boot Manager SVN 也已检查。

本轮又确认：

- Windows 已重新安装，Secure Boot enforcement 开启后故障仍可复现；
- `Enforce Secure Boot = Disabled` 时，内部 Windows 可启动；
- `Enforce Secure Boot = Enabled` 并保存重启后，内部 Windows Boot Manager 仍启动失败；
- 同一状态下，从 Boot Manager 手动选择独立 EFI USB 设备，也出现 `EFI USB Device (VendorCoProductCode) boot failed.`；
- 关闭 Secure Boot enforcement 后，该 USB 可用于正常启动/安装。

因此，“仅当前 Windows 安装、BCD、ESP 路径或单个 `bootmgfw.efi` 损坏”已显著降低优先级。当前更应关注 **固件执行 Secure Boot 验证时的行为、固件变量/数据库的实际运行状态，以及 OEM BIOS/EC 对当前 Microsoft Secure Boot 证书迁移状态的兼容性**。

这个结论仍不等于已经证明“BIOS 二进制文件本身有 bug”。固件代码缺陷与当前 NVRAM Secure Boot 数据库/策略状态异常，尚未被区分。

## 1. 重新安装 Windows 后仍复现

用户已完成一次新的 Windows 安装。随后再次启用 `Enforce Secure Boot`，内部 Windows 仍不能启动。

这意味着重装系统没有改变故障的核心触发条件。由于 Secure Boot 的 PK/KEK/db/dbx 等固件变量不属于普通 Windows 系统分区，重装本身不能视为对这些固件状态的重置。

## 2. BIOS 现场状态

Insyde H2O 页面再次确认设备信息：

- COLORFUL P16 Pro
- CPU：Intel Core i9-13900HX
- BIOS：`1.07.05COLO3`
- KBC/EC：`1.09.04CF1`
- ME FW：`16.1.32.2473`
- Memory：16384 MB，DRAM 5600 MHz

Administer Secure Boot 页面在保存/重启前后可见：

- `Secure Boot Database = Installed and Locked`
- `User Customized Security = NO`
- `Enforce Secure Boot` 可在 Disabled / Enabled 间切换
- `Erase all Secure Boot Settings`
- `Restore Secure Boot to Factory Settings`
- `PK Options`
- `KEK Options`
- `DB Options`
- `DBX Options`
- `DBT Options`
- `DBR Options`
- `Select a UEFI file as trusted for execution`

`Select a UEFI file as trusted for execution` 页面右侧说明为：

`Add specific EFI image hash to allowed database.`

即该功能会把所选 EFI 文件的 hash 加入允许数据库；本轮没有执行该写入。

## 3. 关键新增 A/B：独立 EFI USB 也被 enforcement 拒绝

### Secure Boot enforcement 关闭

- 内部 Windows：可启动
- 用户用于安装 Windows 的 EFI USB：可启动

### Secure Boot enforcement 开启并保存重启

- 内部 Windows Boot Manager：启动失败
- Boot Manager 中可以看到：
  - `Windows Boot Manager (... YMTC PC4...)`
  - `EFI USB Device (VendorCoProductCode)`
- 手动选择 USB 后固件直接显示：

`EFI USB Device (VendorCoProductCode) boot failed.`

这是一条新的关键证据。它说明故障不再只绑定于内部 SSD 上的单一 Windows Boot Manager 路径；另一个独立 EFI 启动入口在 enforcement 开启时也被固件拒绝。

### 解释边界

这组结果强烈降低以下假设的优先级：

- 仅内部 BCD 错误；
- 仅内部 ESP 损坏；
- 仅内部 `bootmgfw.efi` 单文件损坏；
- 仅当前 Windows 安装状态异常。

但它仍不能单独证明：

- 固件二进制代码一定有 bug；
- USB 上的每个 EFI 组件都与当前 db/dbx/SVN 完全兼容；
- 当前 Secure Boot 变量一定未受历史更新影响。

## 4. DB 页面现场内容

进入：

`Administer Secure Boot -> DB Options`

BIOS 直接显示 DB Signature List，共 7 个 PKCS7 条目：

1. `Microsoft Windows Production PCA 2011`
2. `Windows UEFI CA 2023`
3. `Microsoft Corporation UEFI CA 2011`
4. `Microsoft UEFI CA 2023`
5. `Microsoft Option ROM UEFI CA 2023`
6. `Secure Certificate`
7. `Cus CA`

页面同时提供：

- `Enroll Signature`
- `Delete Signature`

选择 Enroll 时可选择签名格式：

- `PKCS7`
- `SHA256`
- `SHA512`

本轮没有新增或删除任何 DB 项。

这个现场观察与 2026-10-04 Windows 侧解析结果一致地说明：**当前 db 不是空的，也不能继续把“单纯缺少 Windows UEFI CA 2023”当作首要解释。**

## 5. DBX 页面现场内容

进入：

`Administer Secure Boot -> DBX Options`

可见：

- `Enroll Signature`
- `Delete Signature`
- 多条 `[SHA256]` 撤销项

页面显示至少前 10 条 SHA-256 值，并有滚动条，说明条目数量更多。

此前 Windows 侧已经完整解析当前 dbx 为 416 条 SHA-256 数据，并对当时检查的 EFI 映像做过比对，未发现命中。

本轮没有删除、覆盖或新增 DBX 项。

## 6. 启动周期限制提示

在某一轮已经尝试启动 boot device 后，再执行部分 Secure Boot 管理操作，固件显示：

`The operation is only allowed before booting any boot device!!! Please reset system and enter this menu directly if need use this operation.`

这说明该 Insyde 固件对部分 Secure Boot 管理动作有启动周期状态限制。需要重新启动，并在尝试任何 boot device 前直接进入相关菜单。

这不是新的 Windows 故障，也不是 USB 故障本身。

## 7. 当前工作假设排序

### 第一优先级

**OEM/Insyde 固件实际执行 Secure Boot 验证时的兼容性或状态问题。**

包括但不限于：

- 固件代码对当前 Microsoft 2011 -> 2023 迁移状态处理异常；
- 固件读取/应用 db、dbx、DBT、DBR 或相关安全变量时存在运行时问题；
- 当前变量组合在 Windows 侧看起来结构有效，但固件实际 enforcement 行为异常；
- 该 BIOS/EC 版本存在 OEM 已知修复。

### 第二优先级

**当前 Secure Boot 固件变量/默认数据库状态需要 OEM 认可的恢复或重新初始化。**

这与“Erase all Secure Boot Settings”不同。是否应执行 `Restore Secure Boot to Factory Settings`，必须先确认本机 BIOS ROM 内置的 factory database 是否包含当前所需 2023 证书，以及 OEM 是否建议这样做。

### 已明显降低优先级

- 单纯缺少 Windows UEFI CA 2023；
- 单纯 BCD/ESP 路径错误；
- 单个内部 `bootmgfw.efi` 损坏；
- 再次重装 Windows；
- 直接运行 BCDBoot；
- 清 TPM。

## 8. 目前不要做

在 OEM 明确答复前，暂不执行：

- `Erase all Secure Boot Settings`
- 手工删除 PK / KEK / db / dbx / DBT / DBR
- 删除 DBX 项
- 手工把内部 `bootmgfw.efi` 加入 trusted hash
- `Restore Secure Boot to Factory Settings`
- Clear TPM
- 再次重装 Windows
- 刷任何未确认适用于本机准确子型号的 BIOS/EC

当前保持 `Enforce Secure Boot = Disabled` 可恢复 Windows 正常使用。

## 9. 下一步：OEM 固件支持

下一步优先联系 COLORFUL，提供以下信息：

- COLORFUL P16 Pro
- Intel Core i9-13900HX
- BIOS `1.07.05COLO3`
- KBC/EC `1.09.04CF1`
- ME FW `16.1.32.2473`

并提供明确 A/B：

- Secure Boot OFF：Windows / EFI USB 都可启动
- Secure Boot ON：Windows Boot Manager / EFI USB 都 boot failed
- Windows 已重新安装，现象不变
- Secure Boot Database 显示 Installed and Locked
- DB 现场可见 Windows UEFI CA 2023、Microsoft UEFI CA 2023 等证书

需要 OEM 明确回答：

1. 是否存在比 `1.07.05COLO3` 更新、且与本机准确子型号匹配的 BIOS；
2. 是否需要配套 KBC/EC 更新；
3. 是否有 Secure Boot / Windows UEFI CA 2023 相关已知问题或修复；
4. 是否建议执行 `Restore Secure Boot to Factory Settings`；
5. 当前 BIOS ROM 的 factory Secure Boot database 是否包含 2023 证书；
6. 是否有官方 Secure Boot database recovery / BIOS recovery 流程。

在取得准确官方固件前，不使用其他 Insyde、相似模具或其他 P16 机型的 BIOS 进行交叉刷写。
