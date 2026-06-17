# Auto-WorkStation

基于 [chezmoi](https://chezmoi.io) 的个人开发环境配置仓库。

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

## 快速开始

### 1. 安装 Homebrew（如未安装）

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. 安装 chezmoi

```bash
brew install chezmoi
```

### 3. 初始化 dotfiles

```bash
chezmoi init git@github.com:l00-ai-workstation/auto-workstation.git
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

## 许可证

MIT

## Documentation

项目长期维护文档：
- [Phase 1 完整实施记录](docs/Phase1.md)—— 详细步骤、踩坑记录、维护流程、灾难恢复、版本升级策略
