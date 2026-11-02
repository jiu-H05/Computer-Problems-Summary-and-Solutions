# Computer Problems: Summary and Solutions

> 文档记录日期：2026-11-02
> 运行环境：Windows 11（主机名 LENOVO），桌面路径 `E:\Desktop`
> 涉及对象：桌面快捷方式（`.lnk`）、Windows 安全区域、强制完整性控制（MIC）

---

## 一、问题概述

桌面出现两类异常，且相互关联：

1. **打开快捷方式时弹出"打开文件 - 安全警告"**，提示"来自 Internet 的文件可能对你有所帮助，但此文件类型可能危害你的计算机"，发行商显示"未知发布者"。
2. **部分快捷方式图标变成空白文档图标**（白色文件样式），无法显示程序原始图标。

---

## 二、问题现象

### 2.1 安全警告弹窗

- 弹窗名称：`E:\Desktop\DSH Desktop.lnk`
- 类型：快捷方式
- 提示内容：来自 Internet 的文件，未知发布者
- 触发范围：每个异常快捷方式打开时均会出现

### 2.2 图标异常

桌面 15 个快捷方式中，有 8 个显示为空白图标：

| 序号 | 快捷方式 | 目标程序 |
|----|----------|----------|
| 1 | 腾讯会议 | `C:\Program Files\Tencent\WeMeet\WeMeetApp.exe` |
| 2 | 夸克网盘 | `D:\QuarkCloudDrive\quark_cloud_drive.exe` |
| 3 | Clash Verge | `D:\Scoop\apps\clash-verge-rev\current\clash-verge.exe` |
| 4 | CC Switch | `D:\Scoop\apps\cc-switch\cc-switch.exe` |
| 5 | ChatGPT | `D:\Scoop\apps\chatgpt\current\ChatGPT.exe` |
| 6 | Firefox | `D:\Scoop\apps\firefox\current\firefox.exe` |
| 7 | Google Chrome | `D:\Scoop\apps\googlechrome\current\chrome.exe` |
| 8 | DSH Desktop | `D:\Deepseek Harness\DSH Desktop\DSH Desktop.exe` |

其余 7 个快捷方式（豆包、Kimi、原神、绝区零、异环、无畏契约、考试安全客户端）图标正常。

---

## 三、诊断过程

### 3.1 排查安全警告

#### 第一步：检查网络来源标记（MOTW）

通过 NTFS 备用数据流检查 `Zone.Identifier`：

```powershell
Get-Item -LiteralPath "E:\Desktop\DSH Desktop.lnk" -Stream *
```

结果：文件只有 `:$DATA` 流，**不存在 `Zone.Identifier`**。目标 exe 同样没有该标记。排除了"文件来自互联网下载标记（MOTW）"这一常规原因。

#### 第二步：检查快捷方式目标

```powershell
$ws = New-Object -ComObject WScript.Shell
$lnk = $ws.CreateShortcut("E:\Desktop\DSH Desktop.lnk")
$lnk.TargetPath        # 目标程序路径
$lnk.IconLocation      # 图标位置
```

目标程序存在且可正常启动，排除快捷方式失效。

#### 第三步：检查安全区域协议默认值

定位注册表 `ZoneMap\ProtocolDefaults`，发现关键问题：

```
HKCU\...\ZoneMap\ProtocolDefaults          => file = 3
HKLM\...\ZoneMap\ProtocolDefaults          => file = 3
HKLM\...\WOW6432Node\...\ProtocolDefaults  => file = 3
```

安全区域编号含义：

| 值 | 区域 |
|----|------|
| 0 | 本地计算机（MyComputer） |
| 1 | 本地 Intranet |
| 2 | 受信任站点 |
| 3 | Internet |
| 4 | 受限制站点 |

`file` 协议被错误设置为 **3（Internet 区域）**，导致系统将所有通过 `file://` 协议访问的本地文件当作 Internet 来源，从而触发安全警告。

### 3.2 排查图标空白

#### 第一步：检查图标引用

发现异常快捷方式的 `IconLocation` 字段为 `,0`（图标路径为空，仅保留索引），属于无效引用；但修正图标路径、重建图标缓存、重启资源管理器后，图标仍然空白。

#### 第二步：检查图标缓存

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\Windows\Explorer" -Filter iconcache*
```

发现多个 `iconcache_*.db` 文件大小为 0 或 24 字节（正常应为数百 KB 至数 MB），确认图标缓存损坏，随后进行了重建。

#### 第三步：检查文件完整性标签（关键）

```powershell
icacls "E:\Desktop\Google Chrome.lnk"
```

发现异常文件带有：

```
Mandatory Label\Low Mandatory Level:(I)(NW)
```

批量比对确认：**8 个空白图标文件全部带有低完整性标签，7 个正常文件均无此标签**，两者完全对应。

#### 第四步：检查桌面目录本身

```powershell
icacls "E:\Desktop"
```

发现桌面文件夹本身被显式标记为低完整性：

```
Mandatory Label\Low Mandatory Level:(OI)(CI)(NW)
```

而其父目录 `E:\` 无此标签，说明该标签是针对桌面文件夹单独设置的。

---

## 四、根因分析

### 4.1 安全警告根因

注册表中 `file` 协议默认安全区域被篡改：由正常值 **0（本地计算机）** 改为 **3（Internet）**，且同时存在于当前用户、本地机器、Wow6432Node 三处。本地文件因此被统一判定为 Internet 来源。

注册表键的访问控制列表（ACL）同时被加固：

- 所有者：`NT SERVICE\TrustedInstaller`
- `BUILTIN\Administrators` 仅有 `ReadKey`（只读）权限
- 仅 `TrustedInstaller` 拥有完全控制

这导致普通管理员无法直接修改，需要先取得所有权。

### 4.2 图标空白根因

Windows 强制完整性控制（MIC）对文件标签遵循"**取低**"原则：

- 在低完整性目录中**新建或复制**文件时，文件完整性级别取"进程级别"与"目录级别"中的**较低者**。即使中完整性的资源管理器在低完整性目录中创建文件，文件也会被强制降为低完整性。
- 这 8 个快捷方式是在桌面内创建或复制进入的，因此被打上低完整性标签，资源管理器无法正常加载其图标。
- 另外 7 个快捷方式是从其他中完整性目录**移动**过来的，同卷移动保留原安全描述符（包括标签），不重新继承目录标签，因此保持中完整性，图标正常。

### 4.3 来源溯源

系统中存在一套基于 Windows 完整性机制的本地沙箱环境：

- 本地组：`CodexSandboxUsers`
- 本地账户：`CodexSandboxOffline`、`CodexSandboxOnline`
- 应用日志中存在 `windowsSandbox`、`codex-sandbox` 等记录

该沙箱属于 ChatGPT/Codex 桌面应用的本地代码执行隔离组件，会将其工作目录设置为低完整性以限制沙箱内代码能力。结合环境证据，高度可能是该沙箱在执行涉及桌面的任务时，将 `E:\Desktop` 标记为低完整性。

### 4.4 附：UAC 窗口一闪而过的原因

注册表 `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` 中：

```
ConsentPromptBehaviorAdmin = 0   （不提示直接提升；默认值为 5）
EnableLUA = 1
```

管理员提权请求被自动批准，因此 UAC 窗口短暂闪现后立即通过，无需手动点击。

---

## 五、解决方案

### 5.1 修复安全区域注册表

#### 当前用户（无需提权）

```powershell
$pd = 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\ProtocolDefaults'
Set-ItemProperty -Path $pd -Name 'file' -Value 0 -Type DWord
```

#### 本地机器（需提权并夺取所有权）

由于键权限被 TrustedInstaller 锁定，修复流程为：

1. 启用特权：`SeTakeOwnershipPrivilege`、`SeRestorePrivilege`、`SeBackupPrivilege`、`SeSecurityPrivilege`
2. 以 `TakeOwnership` 权限打开注册表键，将所有者设为 `Administrators`
3. 授予 `Administrators` 完全控制权限
4. 将 `file` 值修改为 `0`
5. 恢复原始 ACL 与 TrustedInstaller 所有者

关键调用（PowerShell）：

```powershell
# 夺取所有权
$rk = [Microsoft.Win32.Registry]::LocalMachine.OpenSubKey(
    $sub,
    [Microsoft.Win32.RegistryKeyPermissionCheck]::ReadWriteSubTree,
    [System.Security.AccessControl.RegistryRights]::TakeOwnership)
$sec = $rk.GetAccessControl()
$sec.SetOwner((New-Object System.Security.Principal.NTAccount('BUILTIN\Administrators')))
$rk.SetAccessControl($sec)
$rk.Close()
```

> 注意：启用特权时，`TOKEN_PRIVILEGES` 结构体必须使用 `Pack=1`，否则 LUID 偏移错误，会返回错误 1300（ERROR_NOT_ALL_ASSIGNED）。

### 5.2 修复图标完整性标签

#### 重建图标缓存

```powershell
Stop-Process -Name explorer -Force
Remove-Item "$env:LOCALAPPDATA\Microsoft\Windows\Explorer\iconcache_*.db" -Force
Start-Process explorer.exe
ie4uinit.exe -show
```

#### 将 8 个快捷方式恢复为中完整性

```powershell
icacls "E:\Desktop\<快捷方式>.lnk" /setintegritylevel M
```

#### 将桌面目录本身恢复为中完整性（根治，防止复发）

```powershell
icacls "E:\Desktop" /setintegritylevel M
```

---

## 六、验证结果

修复后逐项检查：

| 检查项 | 结果 |
|--------|------|
| HKCU `file` 值 | 0 |
| HKLM `file` 值 | 0 |
| Wow6432Node `file` 值 | 0 |
| 桌面目录完整性标签 | Medium |
| 8 个快捷方式完整性标签 | Medium |
| 低完整性残留 | 无 |
| 桌面图标显示 | 全部恢复正常 |
| 快捷方式打开安全警告 | 不再出现 |

---

## 七、预防建议

1. 谨慎使用系统加固/安全优化类工具，此类工具可能批量修改安全区域、完整性标签、注册表权限和 UAC 策略。
2. 若需恢复 UAC 手动确认，可在本地安全策略中将"用户账户控制：管理员批准模式中管理员的提升权限提示行为"改回"提示同意"（对应值 5）。
3. 定期检查关键注册表项与桌面目录的完整性标签，避免异常复发。
4. 新建的桌面文件已不会再被强制降为低完整性，日常使用无需额外操作。

---

## 八、附录：关键位置速查

| 类别 | 路径 / 标识 |
|------|-------------|
| 安全区域注册表 | `HKCU\...\Internet Settings\ZoneMap\ProtocolDefaults` |
| 安全区域注册表（机器） | `HKLM\...\Internet Settings\ZoneMap\ProtocolDefaults` |
| 低完整性 SID | `S-1-16-4096` |
| 中完整性 SID | `S-1-16-8192` |
| 高完整性 SID | `S-1-16-12288` |
| TrustedInstaller SID | `S-1-5-80-956008885-3418522649-1831038044-1853292631-2271478464` |
| 图标缓存目录 | `%LOCALAPPDATA\Microsoft\Windows\Explorer` |
| 沙箱本地组 | `CodexSandboxUsers` |
| 沙箱账户 | `CodexSandboxOffline` / `CodexSandboxOnline` |
