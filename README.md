# Claude Disk Cleanup Plugin

> Windows 磁盘清理与空间管理 Claude Code 插件

自动分析磁盘占用，安全清理垃圾文件，并将大型缓存/数据目录迁移到其他磁盘。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blue.svg)](https://code.claude.com/docs/en/plugins)

---

## Features

- **智能磁盘分析** — 自动扫描所有磁盘，找出空间占用大户
- **安全清理** — 识别并清理临时文件、缓存、旧安装包等
- **大文件迁移** — 将 HuggingFace/Ollama/WSL/pip/pnpm 缓存迁移到其他磁盘
- **环境变量管理** — 自动设置环境变量，迁移后应用无需重新配置
- **安全优先** — 每步操作前向用户确认，绝不会自动删除重要文件

## Supported Cleanup Targets

| Category | Items | Safe to Delete? |
|----------|-------|----------------|
| **Cache** | pip, pnpm, npm, conda | Yes — will re-download as needed |
| **Temp Files** | `%TEMP%`, IDE cache, launcher tmp | Yes — no impact on functionality |
| **Recycle Bin** | All files | Yes — after user confirmation |
| **Old Installers** | LeStoreDownload, LenovoSoftstore | Yes — already installed apps unaffected |
| **AI Model Cache** | HuggingFace, Ollama | Migrate — set env vars to new path |
| **Docker/WSL** | WSL distributions | Migrate — export/import to new drive |

## Supported Migrations

| Tool | Source | Env Variable | Target |
|------|--------|-------------|--------|
| HuggingFace | `~\.cache\huggingface` | `HF_HOME` | `D:\HuggingFace` |
| Ollama | `~\.ollama\models` | `OLLAMA_MODELS` | `D:\Ollama\models` |
| Docker/WSL | `%LOCALAPPDATA%\Docker\wsl` | N/A | `D:\WSL\` |
| pip | `%LOCALAPPDATA%\pip` | `PIP_CACHE_DIR` | `D:\pip\cache` |
| pnpm | `%LOCALAPPDATA%\pnpm` | `store-dir` | `D:\pnpm\store` |

## Installation

### Method 1: Install from Git (Recommended)

```bash
/plugin install git@github.com:Junsen-Lin/claude-disk-cleanup.git
```

### Method 2: Manual Installation

1. Clone or download this repository:

```bash
git clone https://github.com/Junsen-Lin/claude-disk-cleanup.git
```

2. Copy the plugin to your Claude Code plugins directory:

```bash
# Windows
cp -r claude-disk-cleanup ~/.claude/plugins/

# Or create a symlink
mklink /D "%USERPROFILE%\.claude\plugins\claude-disk-cleanup" "path\to\claude-disk-cleanup"
```

3. Restart Claude Code.

### Method 3: Skill Only (Without Plugin Structure)

If you only want the skill without the full plugin structure:

1. Create the skill directory:

```bash
mkdir -p ~/.claude/skills/disk-cleanup
```

2. Copy `SKILL.md`:

```bash
cp skills/disk-cleanup/SKILL.md ~/.claude/skills/disk-cleanup/
```

## Usage

### Trigger the Skill

Simply tell Claude about your disk problem:

```
"C盘满了，帮我清理"
"磁盘空间不足"
"哪些文件能删"
"把 HuggingFace 缓存移到 D 盘"
"disk cleanup"
```

### Example Workflow

```
User: C盘快满了，帮我看看

Claude: [Runs disk analysis commands]
Here's what I found:

C Drive: 277GB used / 22.7GB free
D Drive: 581GB used / 70.6GB free

Biggest space consumers:
1. HuggingFace cache — 41.4GB (video models + detection models)
2. Docker/WSL — 45.5GB
3. Ollama models — 7.2GB

I can:
1. Delete detection models from HuggingFace cache (~2GB)
2. Migrate Docker/WSL to D drive (~45GB)
3. Migrate HuggingFace cache to D drive (~39GB)
4. Migrate Ollama models to D drive (~7GB)

Which would you like me to do?

User: 迁移 Docker/WSL 和 HuggingFace 到 D 盘

Claude: [Executes migration with user confirmation at each step]
Done! C drive now has 123GB free (was 22.7GB).
```

## Configuration

### Default Target Drive

The plugin defaults to migrating files to the `D:` drive. If your secondary drive has a different letter, the plugin will adapt based on your system's drive configuration.

### Custom Migration Paths

When the plugin asks for confirmation, you can specify custom target paths:

```
User: 把 Ollama 模型迁移到 E:\AI\Models
```

## Safety Rules

The plugin follows strict safety rules:

1. **Never auto-delete user files** — Documents, Desktop, Downloads always require confirmation
2. **Never delete system files** — `C:\Windows`, `C:\Program Files` are off-limits
3. **Create target directory first** — Ensures migration target exists before moving files
4. **Keep old directory shell** — Only empties content, doesn't remove the directory itself
5. **User-level env vars only** — Never uses Machine-level variables to avoid affecting other users
6. **Report space changes** — Shows before/after disk usage after each operation
7. **WSL migration requires confirmation** — Destructive operation (unregister + import)

## Troubleshooting

### HuggingFace models not found after migration

Set the `HF_HOME` environment variable and restart your terminal:

```powershell
[System.Environment]::SetEnvironmentVariable("HF_HOME", "D:\HuggingFace", "User")
```

### Ollama shows empty model list after migration

Restart the Ollama service:

```cmd
taskkill /f /im ollama.exe
ollama serve
```

Then verify with `ollama list`.

### WSL data appears lost after migration

1. Check that the export `.tar` file is complete
2. Verify the import succeeded: `wsl --list --verbose`
3. Keep the backup `.tar` until you confirm everything works

### Cannot delete LeStoreDownload files

Some files are protected by the system. Use an **administrator terminal**:

```cmd
del /q /s D:\LeStoreDownload\*
```

### Environment variable not taking effect

Environment variables set with `[System.Environment]::SetEnvironmentVariable` require a **new terminal session** to take effect. Close and reopen your terminal.

## How It Works

```
┌─────────────────────────────────────────────────┐
│                  User Request                    │
│         "C盘满了" / "clean up disk"              │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Disk Analysis                       │
│  • Scan all drives for usage                    │
│  • Identify large folders (>1GB)                │
│  • Categorize: cache / temp / migrate / keep    │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              User Confirmation                   │
│  • Present findings with sizes                  │
│  • Ask which items to clean/migrate             │
│  • Show estimated space savings                 │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Execution                           │
│  • Clean: delete temp/cache/recycle bin         │
│  • Migrate: move files + set env vars           │
│  • Report: show actual space freed              │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│              Verification                        │
│  • Check disk space before/after                │
│  • Verify apps still work                       │
│  • Provide restart instructions if needed       │
└─────────────────────────────────────────────────┘
```

## Requirements

- **OS**: Windows 10/11
- **Claude Code**: Latest version
- **Optional**: `gh` CLI (for GitHub operations), `wsl` (for Docker/WSL migration)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## Author

**Junsen-Lin** — [GitHub](https://github.com/Junsen-Lin)

---

If this plugin helped you free up disk space, consider giving it a star.
