# Phase 1 — Dotfiles as Code 完整实施记录

本文档是 README.md 的详细补充，记录 Phase 1 的完整实施过程、踩坑经验和长期维护方法。
README.md 面向使用者，提供项目简介和快速开始；本文档面向维护者，记录实施过程、设计决策、踩坑经验、维护方法和排错流程。

## 实际环境

- **硬件**: Apple M2 (arm64)
- **操作系统**: macOS 26.4.1 (Darwin 25)
- **Shell**: zsh 5.9
- **仓库**: `git@github.com:l00-ai-workstation/Auto-WorkStation.git`
- **chezmoi source**: `~/.local/share/chezmoi`
- **完成日期**: 2026-06-17

---

## chezmoi 工作原理（必读）

第一次接触 chezmoi 容易误以为 `~/Auto-WorkStation` 就是 chezmoi 的工作目录。实际上不是。

```text
GitHub 仓库
     ↓
chezmoi init
     ↓
~/.local/share/chezmoi   ← 真正的 source 目录（chezmoi source-path 返回值）
     ↓
chezmoi apply
     ↓
~/.zshrc
~/.gitconfig
~/Library/Application Support/Code/User/settings.json
```

`~/Auto-WorkStation` 只是最初用来组织文件、执行第一次 `git push` 的工作目录。`chezmoi init` 之后，真正生效的是 `~/.local/share/chezmoi`。日常维护配置文件，也应该在这个目录下操作。

---

## 实施步骤（Step 1.1 - 1.7）

### Step 1.1 创建本地仓库

```bash
mkdir ~/Auto-WorkStation
cd ~/Auto-WorkStation
git init
git remote add origin git@github.com:l00-ai-workstation/Auto-WorkStation.git
```

> 注：计划原定仓库名为 `dotfiles`，实际使用了 `Auto-WorkStation`。功能等价。

### Step 1.2 安装 chezmoi

```bash
brew install chezmoi
chezmoi --version  # 验证：v2.70.5
```

> 注：计划原方案是 `sh -c "$(curl -fsLS get.chezmoi.io)"`，实际通过 brew 安装，效果相同。

### Step 1.3 环境审计（额外步骤）

在生成配置文件之前，先让 Claude Code 对当前环境做了完整审计，输出 A-G 七部分报告，导出为 `环境审计报告.docx`。审计结论：

- 三大版本管理器（pyenv/nvm/sdkman）正确隔离运行时
- Homebrew 仅装了必要独立工具，无全局污染
- 主要工作是剥离 `.zshrc` 中的敏感信息

### Step 1.4 生成配置文件（手工审核4轮，共v1-v4）

纳入 chezmoi 管理的文件：

| 文件 | chezmoi 源文件 | 部署目标 |
|------ | --------------- | --------- |
| `.zshrc` | `dot_zshrc.tmpl` | `~/.zshrc` |
| `.gitconfig` | `dot_gitconfig` | `~/.gitconfig` |
| VSCode settings | `Library/Application Support/Code/User/settings.json` | `~/Library/...` |

同时生成仓库自身文件（不部署到 `$HOME`）：`Brewfile`、`README.md`、`.chezmoiignore`

不纳入的文件及原因：

| 文件 | 原因 |
|------ | ------ |
| `.api_keys` | 明文 API Key，严禁入仓 |
| `.claude.json` | 含 userID、会话数据，机器绑定 |
| `.ssh/` `.gnupg/` | 密钥，严禁入仓 |
| `.nvm/` `.pyenv/` `.sdkman/` | 版本管理器运行时数据 |

关键审核修订记录（v1→v4）：

| 版本 | 修订内容 |
|------|---------|
| v1→v2 | `.zshrc` Anthropic 变量加 `if [[ -n "$DEEPSEEK_API_KEY" ]]` 条件守卫；`settings.json` 修复 JSON 括号错误 |
| v2→v3 | `.zshrc` Python 3.9 PATH 加 `[[ -d ]]` 目录存在性检查 |
| v3→v4 | `README.md` 修复 Markdown 代码块未闭合（阻断级）；`.zshrc` pyenv init 加 `command -v pyenv` 存在性守卫 |

### Step 1.5 配置 SSH 并推送

首次配置 SSH 时，需要先将 GitHub 加入 known_hosts，否则 push 会报 `Host key verification failed`：

```bash
mkdir -p ~/.ssh
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

然后生成密钥并添加到 GitHub：

```bash
ssh-keygen -t ed25519 -C "l00lab"
cat ~/.ssh/id_ed25519.pub  # 复制公钥
# GitHub → Settings → SSH and GPG keys → New SSH key → 粘贴
```

验证后推送：

```bash
ssh -T git@github.com  # 应显示：Hi l00lab! You've successfully authenticated...
git add .
git commit -m "Initial Auto-WorkStation setup"
git push -u origin main
```

### Step 1.6 chezmoi init 并应用

```bash
# 将 GitHub 仓库克隆到 ~/.local/share/chezmoi
chezmoi init git@github.com:l00-ai-workstation/Auto-WorkStation.git

# 发现问题：Brewfile、README.md 和 docs/ 会被错误部署到 $HOME
# 原因：chezmoi 会把源目录里没有 dot_ 前缀的文件都视为要部署的 dotfile
# 修复：在 .chezmoiignore 末尾追加排除规则
echo "" >> ~/.local/share/chezmoi/.chezmoiignore
echo "# === 仓库自身文档/配置，不部署到 \$HOME ===" >> ~/.local/share/chezmoi/.chezmoiignore
echo "README.md" >> ~/.local/share/chezmoi/.chezmoiignore
echo "Brewfile" >> ~/.local/share/chezmoi/.chezmoiignore
echo "docs/" >> ~/.local/share/chezmoi/.chezmoiignore

# 将 dot_gitconfig 中的 email 占位符改为真实邮箱
code ~/.local/share/chezmoi/dot_gitconfig

# 备份当前 .zshrc
cp ~/.zshrc ~/.zshrc.bak.$(date +%Y%m%d)

# 确认 diff 只剩三项（.zshrc / .gitconfig / settings.json）
chezmoi diff

# 应用
chezmoi apply

# 把修复推回 GitHub
cd ~/.local/share/chezmoi
git add .chezmoiignore dot_gitconfig
git commit -m "fix: exclude Brewfile/README from deployment; update git email"
git push
```

### Step 1.7 验证

```bash
chezmoi diff                    # 空输出 ✅
git config --global user.name   # Loo ✅
git config --global user.email  # 278935470+l00lab@users.noreply.github.com ✅
echo $ANTHROPIC_BASE_URL        # https://api.deepseek.com/anthropic ✅
ls -la ~/.api_keys              # 权限 600 ✅
pyenv version                   # 输出当前 Python 版本 ✅
alias ds                        # 输出 alias 定义 ✅
```

---

## 当前仓库结构

```text
~/.local/share/chezmoi
├── .chezmoiignore
├── Brewfile                        # 不部署到 $HOME，仅用于 brew bundle
├── README.md                       # 不部署到 $HOME，仅用于 GitHub 展示
├── docs/
│   └── Phase1.md                   # 不部署到 $HOME，本文档
├── dot_gitconfig                   # → ~/.gitconfig
├── dot_zshrc.tmpl                  # → ~/.zshrc
└── Library
    └── Application Support
        └── Code
            └── User
                └── settings.json  # → ~/Library/Application Support/Code/User/settings.json
```

快速查看：

```bash
chezmoi cd && tree -L 5
```

## ~/.zshrc 运行时调度底座（代码快照）

本段截取自 dot_zshrc.tmpl 的核心加载区，记录三个版本管理器的带守卫加载顺序。守卫逻辑（[[ -d ]] 和 command -v）确保在工具未安装时，~/.zshrc 不会报错中断，是新机器首次恢复时的关键容错机制。

```zsh
# ============================================
# 1. pyenv — Python 版本隔离（抢占 PATH 最前端）
# ============================================
export PYENV_ROOT="$HOME/.pyenv"
[[ -d "$PYENV_ROOT/bin" ]] && export PATH="$PYENV_ROOT/bin:$PATH"
if command -v pyenv &>/dev/null; then
    eval "$(pyenv init --path)"   # 接管 PATH（必须最先执行）
    eval "$(pyenv init -)"        # 启用 shell 补全和虚拟环境
fi

# ============================================
# 2. nvm — Node 版本动态切换
# ============================================
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# ============================================
# 3. SDKMAN — Java 环境隔离
# ============================================
export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$SDKMAN_DIR/bin/sdkman-init.sh" ]] && source "$SDKMAN_DIR/bin/sdkman-init.sh"
```

> 设计意图：pyenv 通过 --path 将 shims 目录插入 PATH 最前端，确保 python 命令优先命中用户隔离版本；nvm 和 SDKMAN 使用 \.（或 source）加载自身函数，不额外污染 PATH 前缀，三者形成 pyenv > nvm > sdkman 的互斥优先级（实际生效顺序由 echo $PATH 验证）。

## Phase 1 验收标准

- [x] `chezmoi diff` 为空
- [x] `chezmoi source` 与 GitHub 仓库同步（`cd ~/.local/share/chezmoi && git status` 显示 `nothing to commit, working tree clean`）
- [x] `git push` 成功
- [x] `.zshrc` 已纳入管理
- [x] `.gitconfig` 已纳入管理
- [x] VSCode settings 已纳入管理
- [x] `Brewfile` 已纳入仓库
- [x] API Key 未入仓（`~/.api_keys` 在 `.chezmoiignore` 中）
- [x] SSH Key 未入仓（`~/.ssh/` 在 `.chezmoiignore` 中）
- [x] 新机器恢复流程已整理完成

**Phase 1 状态：DONE ✅**
本阶段实施完成，以 GitHub main 分支对应的 Phase 1 文档版本为准。

---

## 日常维护流程

```bash
# 1. 修改配置（自动打开对应的源文件）
chezmoi edit ~/.zshrc
chezmoi edit ~/.gitconfig

# 2. 预览变更
chezmoi diff

# 3. 应用到本机
chezmoi apply

# 4. 提交并推送到 GitHub
cd ~/.local/share/chezmoi
git add .
git commit -m "描述本次修改"
git push
```

---

## 灾难恢复

误改本机配置、`.zshrc` 损坏、或想重置到仓库状态时：

```bash
# 查看本机与仓库的差异
chezmoi diff

# 将本机配置恢复到仓库状态
chezmoi apply

# 查看当前由 chezmoi 管理的所有文件
chezmoi managed

# 如果仓库本身有更新，先拉取再应用
cd ~/.local/share/chezmoi
git pull
chezmoi apply
```

---

## 版本升级策略

升级运行时版本时遵循以下原则，避免引入不稳定因素：

| 运行时 | 升级策略 |
|--------|---------|
| Python | 仅升级小版本（3.11.x → 3.11.y），跨大版本需充分测试 |
| Node | 仅升级 LTS 版本 |
| Java | 仅升级 Temurin LTS 版本 |

升级流程：

```bash
# 1. 本机验证新版本可用
pyenv install 3.11.x && pyenv global 3.11.x  # 以 Python 为例

# 2. 更新 dotfiles（如 .zshrc 中有版本相关配置）
chezmoi edit ~/.zshrc

# 3. 应用并验证
chezmoi apply
python --version

# 4. 推送到 GitHub
cd ~/.local/share/chezmoi
git add .
git commit -m "chore: upgrade Python to 3.11.x"
git push

# 5. 更新本文档「当前状态快照」中的版本号
```

---

## 坑与经验

### 坑1：Brewfile 和 README.md 会被错误部署到 $HOME

chezmoi 会把源目录里没有 `dot_` 前缀的所有文件都视为要部署到 `$HOME` 的 dotfile。`Brewfile` 和 `README.md` 是仓库自身文档，不应该部署。解决方法是在 `.chezmoiignore` 里排除它们。

### 坑2：文档中的 Markdown 代码块建议在 GitHub 页面验证渲染效果，避免围栏未闭合导致后续内容全部变成代码块

第一个 ` ```bash ` 代码块缺少结束符，导致 GitHub 把后续 7 个步骤全部渲染成代码。经过多轮尝试无法修复，根本原因是在对话框中展示含嵌套代码块的 Markdown 时，渲染器会吃掉部分围栏符号，导致"看起来对了但实际没对"。最终解决方式是让 Claude Code 直接写入临时文件再 `cat` 验证，而不是在对话框或网页编辑器里手动复制粘贴长文本。

### 坑3：chezmoi apply 会整体覆盖 .zshrc

应用前需备份：`cp ~/.zshrc ~/.zshrc.bak.$(date +%Y%m%d)`

### 坑4：dot_gitconfig 用空格缩进，现有 .gitconfig 用 Tab

apply 后缩进方式改变，但 git 两种都认，功能不受影响。

### 坑5：SSH 首次配置需要先加 known_hosts

直接 push 会报 `Host key verification failed`。需要先执行 `ssh-keyscan github.com >> ~/.ssh/known_hosts`。

### 坑6：SDKMAN 刚安装后当前 shell 未加载

新机器装完 SDKMAN 后，直接执行 `sdk` 会报 `command not found`。需要先执行 `source "$HOME/.sdkman/bin/sdkman-init.sh"` 或重新打开终端，再安装 Java 版本。

### 坑7：Claude Code 退出后上下文丢失

按 Ctrl+Z 挂起再 `fg` 恢复可保留上下文。如果已退出，用 `claude --continue` 恢复最近一次会话，或 `claude --resume` 选择历史会话。

### 坑8：本机配置漂移 — dot_gitconfig 长期未同步真实生效的 git 配置

仓库迁移到 `l00-ai-workstation` 组织后，本机 `~/.gitconfig` 的 `user.email` 已手动改为 GitHub noreply 邮箱（`278935470+l00lab@users.noreply.github.com`），但 chezmoi 源文件 `dot_gitconfig` 仍停留在旧的私人邮箱。这意味着任何时候执行 `chezmoi apply`，都会把本机正确的配置覆盖回过时状态。

教训：直接编辑 `~/.gitconfig`（或任何受 chezmoi 管理的文件）而不通过 `chezmoi edit` 修改源文件，会导致"本机配置正确、仓库源文件落后"的隐性风险，且不会立刻报错，只会在下一次 `apply` 时悄悄回退。修复方式：

```bash
sed -i '' 's/email = .*/email = 278935470+l00lab@users.noreply.github.com/' ~/.local/share/chezmoi/dot_gitconfig
chezmoi diff
chezmoi apply   # 如遇 "has changed since chezmoi last wrote it?" 提示，输入 overwrite
cd ~/.local/share/chezmoi
git add dot_gitconfig
git commit -m "fix: sync gitconfig email to noreply address"
git push
```

> 用 `sed` 精确替换单行，比在编辑器里手动改更不容易出错，尤其是第一次使用 `chezmoi edit` 调起的编辑器（可能是 vim）时，容易因为不熟悉操作而误改其他内容。

### 坑9：pyenv 接管失败导致 Python 指向系统版本

新机器恢复后，如果 which python 指向 /usr/bin/python3 而非 ~/.pyenv/shims/python，说明 pyenv init 未正确加载。常见原因是 ~/.zshrc 中遗漏了 eval "$(pyenv init --path)" 或该行被其他 PATH 设置覆盖。

修复方式：确认 dot_zshrc.tmpl 中已包含以下两行（缺一不可）：

```bash
eval "$(pyenv init --path)"   # 必须放在 PATH 修改的最前面
eval "$(pyenv init -)"        # 放在后面，启用补全
```

然后执行 source ~/.zshrc 或重新打开终端。

## 环境初始化关键检查点（Phase 1 补遗）

本部分记录多版本管理器正常工作的验证指标与核心加载逻辑，属于新机器恢复后的必要自检项。

### 1. 核心组件快速验证表

| 组件 | 验证命令 | 健康标志 |
|------|---------|---------|
| pyenv | which python | 路径含 .pyenv/shims，非 /usr/bin/python |
| nvm | which node | 路径含 .nvm/versions |
| SDKMAN | java -version | 回显 Temurin 21.x |
| Homebrew | brew doctor | 输出 "Your system is ready to brew." |
| VSCode + .venv | 打开项目后查看状态栏 | 显示当前项目 .venv 解释器 |
| Claude Code | claude -p "respond with only: OK" | 返回 OK |

此表也是 Phase 3 自动化完成后最终验收的“健康检查（Health Check）”清单。全部 PASS 方可进入开发。

### 2. PATH 加载顺序（决胜细节）

新机器恢复后，终端 PATH 的正确顺序应为：

```text
pyenv shims → nvm/versions → sdkman/candidates → Homebrew → 系统原生
```

对应 ~/.zshrc 中的核心片段（已纳入 dot_zshrc.tmpl，含守卫逻辑）：

```bash
# pyenv（必须分两行写，缺一不可）

export PYENV_ROOT="$HOME/.pyenv"
[[ -d "$PYENV_ROOT/bin" ]] && export PATH="$PYENV_ROOT/bin:$PATH"
if command -v pyenv &>/dev/null; then
    eval "$(pyenv init --path)"   # 接管 PATH
    eval "$(pyenv init -)"        # 启用 shell 补全
fi

# nvm

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# SDKMAN

export SDKMAN_DIR="$HOME/.sdkman"
[[ -s "$SDKMAN_DIR/bin/sdkman-init.sh" ]] && source "$SDKMAN_DIR/bin/sdkman-init.sh"
```

验证标准（执行 echo $PATH）：/Users/xxx/.pyenv/shims 应出现在最前面。

### 3. VSCode 绑定 .venv 的必做动作

首次打开新项目需手动指定一次解释器：

1.Cmd + Shift + P → Python: Select Interpreter

2.选择 ./.venv/bin/python

3.状态栏左下角应显示 .venv 字样

原则：VSCode 只负责调用 pyenv/nvm/sdkman 已安装的运行环境，不负责安装运行环境本身。Python/Node/Java 的安装和版本切换由各版本管理器独立完成。

### 4. 新机器恢复后的自检脚本（快速验证）

```bash
echo "=== Python ===" && python --version && which python
echo "=== Node ===" && node --version && which node
echo "=== Java ===" && java -version && which java
echo "=== Homebrew ===" && brew doctor
```

若路径均指向用户目录（~/.pyenv / ~/.nvm / ~/.sdkman），则环境隔离生效。

### 坑10：~/.bash_profile 中的孤儿配置（历史遗留配置冲突风险）

在 Phase 1 早期手动摸索阶段，曾按照教程在 `~/.bash_profile` 中配置了 pyenv、SDKMAN 和 Homebrew 清华镜像。后来这些配置已完整迁移并优化到 chezmoi 管理的 `dot_zshrc.tmpl` 中，但 `~/.bash_profile` 文件长期未清理，成为“休眠的孤儿配置”。

**2026-06-19 执行最终清理**：

```bash
cp ~/.bash_profile ~/.bash_profile.bak.$(date +%Y%m%d)
rm ~/.bash_profile
```

清理后验证：重新打开终端后，python --version、node --version、java -version、echo $PYENV_ROOT 等输出完全正常，未发生任何回归。
教训：所有 Shell 配置必须统一由 chezmoi 管理，坚决杜绝 .bash_profile、.profile 等多配置文件并存。这类历史遗留配置是长期维护中最容易被忽略的隐性定时炸弹。一旦发现，应立即备份并删除。
此操作完成后，~/.zshrc 已成为唯一且权威的配置入口，彻底消除了双 Shell 配置冲突隐患。

---

## 新电脑恢复流程

```bash
# 1. 安装 Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. 安装 chezmoi（新电脑唯一需要手动安装的工具）
brew install chezmoi

# 3. 一条命令拉取并应用 dotfiles
chezmoi init --apply git@github.com:l00-ai-workstation/Auto-WorkStation.git

# 4. 恢复 Homebrew 软件包
# 说明：chezmoi 已由步骤2安装，brew bundle 遇到重复项会自动跳过
brew bundle --file="$(chezmoi source-path)/Brewfile"

# 5. 恢复各语言运行时
# 说明：步骤4会安装 pyenv/nvm/sdkman 等版本管理器，
# 但不会自动安装具体语言版本，仍需手动执行。
# 注意：SDKMAN 刚装完后当前 shell 尚未加载，需先 source 或重新打开终端。
source "$HOME/.sdkman/bin/sdkman-init.sh"
pyenv install <Python版本> && pyenv global <Python版本> && python --version
nvm install --lts
sdk install java 21.0.11-tem

# 6. 恢复 API Key（手动创建，永不入仓）
echo 'export DEEPSEEK_API_KEY="sk-your-key-here"' > ~/.api_keys
chmod 600 ~/.api_keys

# 7. 配置 SSH Key（参见 Step 1.5）
# 8. 执行健康检查（参见"环境初始化关键检查点"中的验证表）
```

---

## 当前成果边界（2026-06-17）

**已实现：**

- ✅ Shell 配置恢复（`.zshrc`）
- ✅ Git 配置恢复（`.gitconfig`）
- ✅ VSCode 配置恢复（`settings.json`）
- ✅ Homebrew 软件清单恢复（`Brewfile`）
- ✅ API Key 与 SSH Key 安全隔离
- ✅ 新电脑一条命令恢复基础工作站配置

**尚未实现（阶段2以后）：**

- ❌ Python / Node / Java 版本自动安装
- ❌ Claude Code 自动配置（`npm install -g @anthropic-ai/claude-code` 及 DeepSeek endpoint 设置）
- ❌ Ollama 模型自动拉取（`ollama pull`）
- ❌ VSCode 扩展自动安装

> 当前完成的是 **Workstation as Code Level 1——Dotfiles as Code**，而不是 Full Bootstrap。工作站配置已固化，开发环境自动引导（Bootstrap）和项目模板化（Project Scaffolding）属于后续阶段。

---

## 当前状态快照（2026-06-17）

| 项目 | 状态 |
|------|------|
| Git 仓库 | `git@github.com:l00-ai-workstation/Auto-WorkStation.git` |
| 最新 commit | 以 GitHub main 分支最新提交为准 |
| chezmoi | v2.70.5 |
| Python | 3.11.9 (pyenv) |
| Node | v24.16.0 (nvm) |
| Java | Temurin 21.0.11 (SDKMAN) |
| VSCode | 执行 `code --version` 后填入实际值 |
| Claude Code | v2.1.175，已配置 DeepSeek Anthropic Endpoint |
| Ollama（`ollama list` 输出） | deepseek-r1:8b / qwen3-vl:8b / qwen2.5:7b |
| API Key | `~/.api_keys`，权限 600，永不入仓 |
| SSH Key | ed25519，已添加到 GitHub |
| Git 用户邮箱 | `278935470+l00lab@users.noreply.github.com`（GitHub noreply，已与 dot_gitconfig 同步） |
| 真实 PATH 优先级（2026-06-19 验证） | /Users/loo/.pyenv/shims → /Users/loo/.nvm/versions/node/v24.16.0/bin → /Users/loo/.sdkman/candidates/java/current/bin → /opt/homebrew/bin → /usr/bin ✅ |

## PATH 优先级链条（可视化）

新机器恢复后，终端在执行任何命令时，PATH 的查找顺序如下：

```text
第 1 优先  →  /Users/xxx/.pyenv/shims              # Python（用户隔离）
第 2 优先  →  /Users/xxx/.nvm/versions/node/v24.x/bin  # Node（用户隔离）
第 3 优先  →  /Users/xxx/.sdkman/candidates/java/current/bin  # Java（用户隔离）
第 4 优先  →  /opt/homebrew/bin                     # Homebrew（系统级工具）
第 5 优先  →  /usr/bin                              # macOS 系统原生
```

这个顺序确保了 python、node、java 命令永远命中用户隔离的版本，而非系统自带版本。换机器时，若 which python 返回 /usr/bin/python，说明 PATH 加载顺序被破坏，应优先检查 ~/.zshrc 中的加载顺序。

## 系统架构总览

以下为当前工作站各层组件的关系图，后续 Phase 2-4 的所有扩展都将围绕此架构进行：

```text
macOS (Darwin 25)
│
├── Homebrew (包管理基础设施)
│   │
│   ├── pyenv        ← Python 版本管理器
│   ├── nvm          ← Node 版本管理器
│   ├── sdkman       ← Java 版本管理器
│   ├── chezmoi      ← Dotfiles 管理核心
│   ├── ollama       ← 本地模型服务
│   └── (其他 Formula / Cask)
│
├── pyenv
│   └── Python 3.11.9
│       └── .venv (项目级虚拟环境，由 uv 或 venv 创建)
│
├── nvm
│   └── Node v24.16.0 (LTS)
│
├── sdkman
│   └── Java Temurin 21.0.11 (LTS)
│
└── VSCode (开发界面工作台)
    │
    ├── 调用 pyenv 提供的 Python 解释器
    ├── 调用 nvm 提供的 Node 运行时
    ├── 调用 sdkman 提供的 Java 运行时
    └── 通过 extensions.json 管理插件
核心设计原则：VSCode 是“工作台”，只负责调用，不负责安装。所有运行时（Python/Node/Java）的安装和版本切换由各版本管理器独立完成，互不干扰。

## Known Limitations

Phase1 不负责：

- 安装 Python
- 安装 Node
- 安装 Java
- 安装 Claude
- 安装 Ollama

仅负责配置恢复。

## Core Principles

1. 配置进入 Git
2. 密钥永不入仓
3. 运行时和配置分离
4. 项目环境优先于全局环境
5. 所有恢复流程可重复执行（Idempotent）

---

## 待完成阶段

### 阶段2：AI Project Template（下一优先级）

**背景：** 换电脑 1~3 年一次，新建项目每周甚至每天。项目模板的 ROI 远高于继续优化工作站恢复流程。

目标：新项目自动带 `.venv` / Python版本 / Node版本 / Java版本 / VSCode配置 / Claude Code 项目规则。

技术路线待定：A. Copier　B. Cookiecutter　C. 自定义脚本。当前倾向 Copier。

完成标志：执行 `copier copy gh:l00-ai-workstation/ai-project-template myapp` 一条命令即可生成包含完整运行时配置的新项目。

### 阶段3：Bootstrap Automation

目标：新电脑执行 `chezmoi init --apply` 后，通过 `run_once` 脚本自动完成以下所有安装，用户仅需补充 SSH Key 和 API Key 即可进入开发状态：

- pyenv 安装 Python 3.11.9
- nvm 安装 Node LTS
- SDKMAN 安装 Java 21
- Claude Code 安装及 DeepSeek endpoint 配置
- Ollama 安装及模型拉取
- VSCode 扩展安装

### 阶段4：Copier 工作流完善

目标：在阶段2基础上引入交互式项目生成，支持 `project_name` / `python_version` / `node_version` 等参数自定义，进一步打磨模板细节。

---

## Document History

| Version | Date | Notes |
| --------- | ------ | ------- |
| v1.0 | 2026-06-17 | Phase 1 documentation finalized |
| v1.1 | 2026-06-17 | 仓库迁移至 l00-ai-workstation 组织；同步 git 邮箱为 noreply 地址；修复仓库结构代码块围栏丢失问题 |
| v1.2 | 2026-06-19 | 补充环境初始化关键检查点（PATH 加载顺序、VSCode 绑定 .venv、自检脚本）；新增坑9（pyenv 接管失败） |
| v1.3 | 2026-06-19 | 新增 ~/.zshrc 运行时调度底座代码快照（含守卫逻辑）；将真实 PATH 顺序验证结果冻结至状态快照表格 |
| v1.4 | 2026-06-19 | 将“环境初始化关键检查点”小节精简重构为结构化验证表，删除冗余科普内容，保持冻结文档定位 |
| v1.5 | 2026-06-19 | 新增“PATH 优先级链条（可视化）”和“系统架构总览”两个章节；在 VSCode 小节补充一句原则说明；在健康检查表增加“全部 PASS 方可进入开发”的注释；新电脑恢复流程中增加 Step 8（健康检查引用） |

Phase 1 至此完成并冻结。
除事实性错误修正外，本文档不再记录新的功能变更。
后续所有新增能力、自动化流程及架构演进统一记录于 Phase 2 及后续文档。
