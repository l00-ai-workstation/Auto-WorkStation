# Auto-WorkStation

一个基于 chezmoi 的 macOS AI 开发工作站初始化仓库。

它可以帮助你在新 Mac 上快速恢复一致的开发环境，包括：

- Homebrew
- Git
- Zsh
- VS Code
- Python（pyenv）
- Node.js（nvm）
- Java（SDKMAN!）
- API Key 管理
- AI 开发工具配置

## Quick Start

首次使用，仅需几条命令即可完成开发环境初始化：

```bash
brew install chezmoi
chezmoi init https://github.com/l00-ai-workstation/Auto-WorkStation.git
chezmoi apply
```

## Features

- 🍺 Homebrew 软件恢复
- 🐍 Python 环境（pyenv）
- 🟢 Node.js 环境（nvm）
- ☕ Java 环境（SDKMAN!）
- ⚙️ Git 配置同步
- 💻 VS Code 配置同步
- 🔑 API Key 本地隔离

## 适用环境

- **硬件:** Apple Silicon (arm64)
- **操作系统:** macOS 26.x (Darwin 25)
- **Shell:** zsh

## 管理范围

| 层级 | 内容 | 文件 |
|------|------|------|
| Shell | `.zshrc` — 别名、PATH、版本管理器集成 | `dot_zshrc.tmpl` |
| Git | `.gitconfig` — 用户身份、别名、默认分支 | `dot_gitconfig` |
| 编辑器 | VSCode `settings.json` — Python + 终端 | `Library/.../settings.json` |
| 包管理 | Homebrew 完整恢复清单 | `Brewfile` |

## 安装与配置

### 1. 安装 Homebrew（如未安装）

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. 安装 chezmoi

```bash
brew install chezmoi
```

### 3. 初始化 dotfiles

推荐使用 HTTPS（无需配置 SSH）：

```bash
chezmoi init https://github.com/l00-ai-workstation/Auto-WorkStation.git
```

如果已经配置了 GitHub SSH Key，也可以使用：

```bash
chezmoi init git@github.com:l00-ai-workstation/Auto-WorkStation.git
```

### 4. 预览变更

```bash
chezmoi diff
```

### 5. 应用配置

```bash
chezmoi apply
```

### 6. 恢复 Homebrew 软件包

```bash
brew bundle --file="$(chezmoi source-path)/Brewfile"
```

### 7. 安装版本管理器运行时

#### nvm (Node.js)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

#### SDKMAN! (Java)

```bash
curl -s "https://get.sdkman.io" | bash
```

### 8. 恢复 API Key

创建 `~/.api_keys`：

```bash
export DEEPSEEK_API_KEY="sk-your-key-here"
```

该文件已在 `.chezmoiignore` 中排除，不会被提交到仓库。

## .zshrc 启动链路

```text
shell 启动
  ├── 加载 AI 模型别名（ds / mor1 / moqw3 / moqw25）
  ├── source ~/.api_keys
  │     └── 注入 DEEPSEEK_API_KEY
  ├── 设置 Anthropic/DeepSeek 环境变量（仅当 KEY 存在时）
  ├── 配置 Homebrew 清华镜像
  ├── 初始化 pyenv
  │     └── Python 版本隔离
  ├── 初始化 nvm
  │     └── Node.js 版本隔离
  └── 初始化 SDKMAN!
        └── Java 版本隔离（含 auto_env）
```

## 安全原则

| 风险等级 | 措施 |
|----------|------|
| 🔴 API Key | `~/.api_keys` 本地文件，`.chezmoiignore` 排除，永不入仓 |
| 🟠 Claude Code | `.claude.json` 不入仓（含 userID、API Key 哈希等信息） |
| 🟡 用户身份 | `.gitconfig` 中 email 为新机手动修改项 |
| 🟢 版本管理器 | `.nvm/`、`.pyenv/`、`.sdkman/` 运行时数据不入仓 |

## Documentation

项目维护文档：

- [Phase 1 完整实施记录](docs/Phase%201%20%E2%80%94%20Dotfiles%20as%20Code.md)
  - 实施步骤
  - 踩坑记录
  - 维护流程
  - 灾难恢复
  - 版本升级策略

## 许可证

MIT
