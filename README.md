# Computer Problems: Summary and Solutions

> Windows 桌面故障排查与修复记录：快捷方式安全警告 + 图标空白问题

[![Platform](https://img.shields.io/badge/Platform-Windows%2011-0078D4)](https://github.com/jiu-H05/Computer-Problems-Summary-and-Solutions)
[![Date](https://img.shields.io/badge/Date-2026--11--02-success)](https://github.com/jiu-H05/Computer-Problems-Summary-and-Solutions)
[![License](https://img.shields.io/badge/Docs-Reference-orange)](https://github.com/jiu-H05/Computer-Problems-Summary-and-Solutions)

## 项目简介

本仓库记录了一次 Windows 桌面环境故障的完整排查与修复过程。故障表现为两类相互关联的异常：打开快捷方式时弹出"来自 Internet 的文件"安全警告，以及部分快捷方式图标显示为空白。文档梳理了从现象、诊断、根因到解决方案与验证的完整链路，可作为同类问题的参考。

## 问题速览

| 问题 | 直接根因 | 修复方式 |
|------|----------|----------|
| 快捷方式打开弹安全警告 | 注册表 `file` 协议安全区域被改为 `3`（Internet），正常值为 `0` | 夺取 TrustedInstaller 所有权后改回 `0` |
| 8 个快捷方式图标空白 | 桌面目录及文件被标记为低完整性级别（Low Mandatory Level） | `icacls /setintegritylevel M` 恢复为中完整性 |

## 根因摘要

- **安全区域被篡改**：`ZoneMap\ProtocolDefaults` 下的 `file` 值在当前用户、本地机器、Wow6432Node 三处均被设为 `3`，且键权限被 TrustedInstaller 锁定，管理员仅有只读权限。
- **强制完整性控制（MIC）**：Windows 对文件标签遵循"取低"原则。在低完整性目录中新建或复制的文件会被强制降级，导致资源管理器无法正常加载图标；通过移动进入的文件则保留原标签。
- **来源溯源**：系统中的 ChatGPT/Codex 本地沙箱（`CodexSandboxUsers`、`CodexSandboxOffline/Online`）基于 MIC 做代码执行隔离，高度可能是其将桌面目录标记为低完整性。

## 核心修复命令

```powershell
# 1. 恢复 file 协议为本地计算机区域
Set-ItemProperty -Path 'HKCU:\...\ZoneMap\ProtocolDefaults' -Name 'file' -Value 0 -Type DWord

# 2. 重建图标缓存
Stop-Process -Name explorer -Force
Remove-Item "$env:LOCALAPPDATA\Microsoft\Windows\Explorer\iconcache_*.db" -Force
Start-Process explorer.exe

# 3. 恢复桌面目录与快捷方式为中完整性
icacls "E:\Desktop" /setintegritylevel M
icacls "E:\Desktop\<快捷方式>.lnk" /setintegritylevel M
```

## 仓库结构

```
Computer-Problems-Summary-and-Solutions/
├── README.md
└── Computer Problems: Summary and Solutions.md   # 完整排查文档
```

## 文档导航

- 完整文档：[Computer Problems: Summary and Solutions.md](./Computer%20Problems:%20Summary%20and%20Solutions.md)

文档包含八个部分：问题概述、问题现象、诊断过程、根因分析、解决方案、验证结果、预防建议、关键位置速查附录。

## 环境信息

| 项目 | 信息 |
|------|------|
| 操作系统 | Windows 11 |
| 主机名 | LENOVO |
| 桌面路径 | `E:\Desktop` |
| 文件系统 | NTFS |
| 记录日期 | 2026-11-02 |

## 关键标识速查

| 标识 | 值 |
|------|----|
| 低完整性 SID | `S-1-16-4096` |
| 中完整性 SID | `S-1-16-8192` |
| TrustedInstaller SID | `S-1-5-80-956008885-3418522649-1831038044-1853292631-2271478464` |
| 错误 1300 | ERROR_NOT_ALL_ASSIGNED（特权结构体需 `Pack=1`） |

## 预防建议

谨慎使用系统加固或安全优化类工具，此类工具可能批量修改安全区域、完整性标签、注册表权限与 UAC 策略；定期检查关键注册表项与桌面目录的完整性标签，可有效避免问题复发。
