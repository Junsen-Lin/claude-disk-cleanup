---
name: disk-cleanup
description: 磁盘清理和空间管理工具。当用户说"磁盘爆满"、"清理磁盘"、"磁盘空间不足"、"C盘满了"、"D盘满了"、"释放空间"、"哪些文件能删"、"移动文件到D盘"、"磁盘清理"、"disk cleanup"时触发。自动分析磁盘占用，识别可安全删除的文件，并执行清理或迁移操作。
version: 1.0.0
---

# 磁盘清理 (Disk Cleanup)

自动分析磁盘空间占用，安全清理垃圾文件，或将大型缓存/数据目录迁移到其他磁盘。

## 工作流程

```
1. 分析 → 2. 确认 → 3. 执行 → 4. 验证
```

每次操作前必须向用户确认，不得自动删除任何文件。

## 第 1 步：磁盘分析

运行以下命令获取全局视图：

```powershell
# 所有磁盘概览
Get-PSDrive -PSProvider FileSystem | Select-Object Name,@{N='UsedGB';E={[math]::Round($_.Used/1GB,2)}},@{N='FreeGB';E={[math]::Round($_.Free/1GB,2)}},@{N='TotalGB';E={[math]::Round(($_.Used+$_.Free)/1GB,2)}} | Format-Table -AutoSize
```

然后深入用户目录找出大文件：

```powershell
# 用户目录 Top 15
Get-ChildItem "$env:USERPROFILE" -Directory -EA SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -File -EA SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    [PSCustomObject]@{ Path = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Select-Object -First 15 | Format-Table -AutoSize

# AppData\Local Top 15
Get-ChildItem "$env:LOCALAPPDATA" -Directory -EA SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -File -EA SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    if ($size -gt 500MB) {
        [PSCustomObject]@{ Path = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
    }
} | Sort-Object SizeGB -Descending | Format-Table -AutoSize

# AppData\Roaming Top 10
Get-ChildItem "$env:APPDATA" -Directory -EA SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -File -EA SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    if ($size -gt 500MB) {
        [PSCustomObject]@{ Path = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
    }
} | Sort-Object SizeGB -Descending | Format-Table -AutoSize

# 非系统盘（如 D 盘）大文件夹
Get-ChildItem "D:\" -Directory -EA SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -File -EA SilentlyContinue | Measure-Object -Property Length -Sum).Sum
    [PSCustomObject]@{ Path = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Format-Table -AutoSize
```

## 第 2 步：分类识别

将扫描结果分为以下类别，向用户展示：

### 可安全清理（不需确认直接删除）
| 类型 | 路径模式 | 说明 |
|------|---------|------|
| pip 缓存 | `$env:LOCALAPPDATA\pip\cache` | 设置 PIP_CACHE_DIR 后清理 |
| pnpm store | `$env:LOCALAPPDATA\pnpm\store` | 设置 store-dir 后清理 |
| 回收站 | 回收站 | `Clear-RecycleBin -Force` |
| 用户临时文件 | `$env:TEMP` | 可直接删除 |
| IDE 缓存 | `.lingma\cache`, `.trae-cn\cache` | 删除不影响功能 |
| 旧安装包 | `D:\LeStoreDownload\*` | .exe 安装包，已安装的软件不受影响 |
| 联想商店缓存 | `D:\LenovoSoftstore\` | 不影响已安装程序 |
| 电脑管家迁移 | `D:\电脑管家迁移文件\` | 旧迁移数据 |

### 可迁移（需设置环境变量）
| 类型 | 源路径 | 环境变量 | 目标路径 |
|------|--------|---------|---------|
| HuggingFace 缓存 | `~\.cache\huggingface` | `HF_HOME` | `D:\HuggingFace` |
| Ollama 模型 | `~\.ollama\models` | `OLLAMA_MODELS` | `D:\Ollama\models` |
| Docker/WSL | `%LOCALAPPDATA%\Docker\wsl` | N/A | `D:\WSL\` |
| pip 缓存 | `%LOCALAPPDATA%\pip` | `PIP_CACHE_DIR` | `D:\pip\cache` |
| pnpm store | `%LOCALAPPDATA%\pnpm` | `store-dir` (pnpm config) | `D:\pnpm\store` |

### 需要用户确认
| 类型 | 说明 |
|------|------|
| 用户文档/桌面文件 | 可能是重要数据 |
| 游戏/大型软件 | 删除后需要重装 |
| conda/anaconda 环境 | 删除后 Python 环境丢失 |

### 不可动
| 类型 | 说明 |
|------|------|
| Windows 系统文件 | `C:\Windows` |
| Program Files 中已安装程序 | 除非用户明确要求卸载 |

## 第 3 步：执行清理

### 3.1 直接清理（用户确认后）

```powershell
# 清理 pip 缓存
pip cache purge

# 清空回收站
Clear-RecycleBin -Force -EA SilentlyContinue

# 清理用户临时文件
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -EA SilentlyContinue

# 清理 IDE 缓存
Remove-Item -Recurse -Force "$env:USERPROFILE\.lingma\cache\*" -EA SilentlyContinue
Remove-Item -Recurse -Force "$env:USERPROFILE\.trae-cn\cache\*" -EA SilentlyContinue

# 清理 XIAN_Web_Launcher tmp
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\XIAN_Web_Launcher\tmp\*" -EA SilentlyContinue
```

### 3.2 迁移 HuggingFace 缓存

```powershell
# 创建目标目录
New-Item -ItemType Directory -Path "D:\HuggingFace" -Force

# 移动现有缓存
Move-Item "$env:USERPROFILE\.cache\huggingface\hub" "D:\HuggingFace\hub" -Force
Move-Item "$env:USERPROFILE\.cache\huggingface\xet" "D:\HuggingFace\xet" -EA SilentlyContinue

# 设置环境变量
[System.Environment]::SetEnvironmentVariable("HF_HOME", "D:\HuggingFace", "User")
```

### 3.3 迁移 Ollama 模型

```powershell
New-Item -ItemType Directory -Path "D:\Ollama" -Force
Move-Item "$env:USERPROFILE\.ollama\models" "D:\Ollama\models" -Force
[System.Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "D:\Ollama\models", "User")
```

### 3.4 迁移 Docker/WSL

```powershell
# 关闭 WSL
wsl --shutdown

# 导出每个发行版
wsl --list --verbose  # 查看发行版列表
wsl --export <名称> "D:\WSL_Backup\<名称>.tar"

# 注销原发行版
wsl --unregister <名称>

# 导入到 D 盘
wsl --import <名称> "D:\WSL\<名称>" "D:\WSL_Backup\<名称>.tar"

# 删除备份
Remove-Item "D:\WSL_Backup\<名称>.tar"

# 清理旧数据
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Docker\wsl" -EA SilentlyContinue
```

### 3.5 迁移 pip 缓存

```powershell
New-Item -ItemType Directory -Path "D:\pip\cache" -Force
[System.Environment]::SetEnvironmentVariable("PIP_CACHE_DIR", "D:\pip\cache", "User")
pip cache purge
```

### 3.6 迁移 pnpm store

```powershell
pnpm config set store-dir "D:\pnpm\store"
```

## 第 4 步：验证

每步完成后检查磁盘空间变化：

```powershell
Get-PSDrive C,D | Select-Object Name,@{N='UsedGB';E={[math]::Round($_.Used/1GB,2)}},@{N='FreeGB';E={[math]::Round($_.Free/1GB,2)}} | Format-Table -AutoSize
```

重启后验证：
- `wsl --list --verbose` — WSL 发行版正常
- `ollama list` — Ollama 模型正常
- `pip cache info` — 缓存路径指向 D 盘
- `pnpm store path` — store 路径指向 D 盘

## 安全规则

1. **永远不自动删除用户文件** — 文档、桌面、下载等必须用户确认
2. **永远不删系统文件** — `C:\Windows`、`C:\Program Files` 除非用户明确要求卸载
3. **迁移前先创建目标目录** — 确保目标路径存在
4. **迁移后保留旧目录的空壳** — 不要删除旧目录本身，只清空内容
5. **环境变量用 User 级别** — 不要用 Machine 级别，避免影响其他用户
6. **每步都报告空间变化** — 让用户看到清理效果
7. **WSL 迁移需要用户确认** — 这是破坏性操作（unregister + import）

## 常见问题

**Q: 移动 HuggingFace 缓存后模型找不到？**
A: 设置 `HF_HOME` 环境变量后需要重启终端。旧缓存目录可以删除。

**Q: Ollama 迁移后 `ollama list` 为空？**
A: 需要重启终端或重启 Ollama 服务（`taskkill /f /im ollama.exe && ollama serve`）。

**Q: WSL 迁移后数据丢失？**
A: 确保导出的 .tar 文件完整，导入成功后再删除旧数据。建议保留备份直到确认一切正常。

**Q: LeStoreDownload 文件无法删除？**
A: 某些文件被系统保护，需要管理员权限。在管理员终端中运行 `del /q /s D:\LeStoreDownload\*`。
