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
GitHub 仓库
↓
chezmoi init
↓
~/.local/share/chezmoi ← 真正的 source 目录（chezmoi source-path 返回值）
↓
chezmoi apply
↓
~/.zshrc
~/.gitconfig
~/Library/Application Support/Code/User/settings.json

text

`~/Auto-WorkStation` 只是最初用来组织文件、执行第一次 `git push` 的工作目录。`chezmoi init` 之后，真正生效的是 `~/.local/share/chezmoi`。日常维护配置文件，也应该在这个目录下操作。

---

## 实施步骤（Step 1.1 - 1.7）

### Step 1.1 创建本地仓库

```bash
mkdir ~/Auto-WorkStation
cd ~/Auto-WorkStation
git init
git remote add origin git@github.com:l00-ai-workstation/Auto-WorkStation.git
注：计划原定仓库名为 dotfiles，实际使用了 Auto-WorkStation。功能等价。

Step 1.2 安装 chezmoi
bash
brew install chezmoi
chezmoi --version  # 验证：v2.70.5
注：计划原方案是 sh -c "$(curl -fsLS get.chezmoi.io)"，实际通过 brew 安装，效果相同。

Step 1.3 环境审计（额外步骤）
在生成配置文件之前，先让 Claude Code 对当前环境做了完整审计，输出 A-G 七部分报告，导出为 环境审计报告.docx。审计结论：

三大版本管理器（pyenv/nvm/sdkman）正确隔离运行时

Homebrew 仅装了必要独立工具，无全局污染

主要工作是剥离 .zshrc 中的敏感信息

Step 1.4 生成配置文件（手工审核4轮，共v1-v4）
纳入 chezmoi 管理的文件：

文件	chezmoi 源文件	部署目标
.zshrc	dot_zshrc.tmpl	~/.zshrc
.gitconfig	dot_gitconfig	~/.gitconfig
VSCode settings	Library/Application Support/Code/User/settings.json	~/Library/...
同时生成仓库自身文件（不部署到 $HOME）：Brewfile、README.md、.chezmoiignore

不纳入的文件及原因：

文件	原因
.api_keys	明文 API Key，严禁入仓
.claude.json	含 userID、会话数据，机器绑定
.ssh/ .gnupg/	密钥，严禁入仓
.nvm/ .pyenv/ .sdkman/	版本管理器运行时数据
关键审核修订记录（v1→v4）：

版本	修订内容
v1→v2	.zshrc Anthropic 变量加 if [[ -n "$DEEPSEEK_API_KEY" ]] 条件守卫；settings.json 修复 JSON 括号错误
v2→v3	.zshrc Python 3.9 PATH 加 [[ -d ]] 目录存在性检查
v3→v4	README.md 修复 Markdown 代码块未闭合（阻断级）；.zshrc pyenv init 加 command -v pyenv 存在性守卫
Step 1.5 配置 SSH 并推送
首次配置 SSH 时，需要先将 GitHub 加入 known_hosts，否则 push 会报 Host key verification failed：

bash
mkdir -p ~/.ssh
ssh-keyscan github.com >> ~/.ssh/known_hosts
然后生成密钥并添加到 GitHub：

bash
ssh-keygen -t ed25519 -C "l00lab"
cat ~/.ssh/id_ed25519.pub  # 复制公钥
# GitHub → Settings → SSH and GPG keys → New SSH key → 粘贴
验证后推送：

bash
ssh -T git@github.com  # 应显示：Hi l00lab! You've successfully authenticated...
git add .
git commit -m "Initial Auto-WorkStation setup"
git push -u origin main
Step 1.6 chezmoi init 并应用
bash
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
Step 1.7 验证
bash
chezmoi diff                    # 空输出 ✅
git config --global user.name   # Loo ✅
git config --global user.email  # luzhenyuat@gmail.com ✅
echo $ANTHROPIC_BASE_URL        # https://api.deepseek.com/anthropic ✅
ls -la ~/.api_keys              # 权限 600 ✅
pyenv version                   # 输出当前 Python 版本 ✅
alias ds                        # 输出 alias 定义 ✅
当前仓库结构
text
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
快速查看：

bash
chezmoi cd && tree -L 5
Phase 1 验收标准
chezmoi diff 为空

chezmoi source 与 GitHub 仓库同步（cd ~/.local/share/chezmoi && git status 显示 nothing to commit, working tree clean）

git push 成功

.zshrc 已纳入管理

.gitconfig 已纳入管理

VSCode settings 已纳入管理

Brewfile 已纳入仓库

API Key 未入仓（~/.api_keys 在 .chezmoiignore 中）

SSH Key 未入仓（~/.ssh/ 在 .chezmoiignore 中）

新机器恢复流程已整理完成

Phase 1 状态：DONE ✅
本阶段实施完成，以 GitHub main 分支对应的 Phase 1 文档版本为准。

日常维护流程
bash
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
灾难恢复
误改本机配置、.zshrc 损坏、或想重置到仓库状态时：

bash
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
版本升级策略
升级运行时版本时遵循以下原则，避免引入不稳定因素：

运行时	升级策略
Python	仅升级小版本（3.11.x → 3.11.y），跨大版本需充分测试
Node	仅升级 LTS 版本
Java	仅升级 Temurin LTS 版本
升级流程：

bash
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
坑与经验
坑1：Brewfile 和 README.md 会被错误部署到 $HOME
chezmoi 会把源目录里没有 dot_ 前缀的所有文件都视为要部署到 $HOME 的 dotfile。Brewfile 和 README.md 是仓库自身文档，不应该部署。解决方法是在 .chezmoiignore 里排除它们。

坑2：文档中的 Markdown 代码块建议在 GitHub 页面验证渲染效果，避免围栏未闭合导致后续内容全部变成代码块
第一个 ```bash 代码块缺少结束符，导致 GitHub 把后续 7 个步骤全部渲染成代码。经过多轮尝试无法修复，根本原因是在对话框中展示含嵌套代码块的 Markdown 时，渲染器会吃掉部分围栏符号，导致"看起来对了但实际没对"。最终解决方式是让 Claude Code 直接写入临时文件再 cat 验证。

坑3：chezmoi apply 会整体覆盖 .zshrc
应用前需备份：cp ~/.zshrc ~/.zshrc.bak.$(date +%Y%m%d)

坑4：dot_gitconfig 用空格缩进，现有 .gitconfig 用 Tab
apply 后缩进方式改变，但 git 两种都认，功能不受影响。

坑5：SSH 首次配置需要先加 known_hosts
直接 push 会报 Host key verification failed。需要先执行 ssh-keyscan github.com >> ~/.ssh/known_hosts。

坑6：SDKMAN 刚安装后当前 shell 未加载
新机器装完 SDKMAN 后，直接执行 sdk 会报 command not found。需要先执行 source "$HOME/.sdkman/bin/sdkman-init.sh" 或重新打开终端，再安装 Java 版本。

坑7：Claude Code 退出后上下文丢失
按 Ctrl+Z 挂起再 fg 恢复可保留上下文。如果已退出，用 claude --continue 恢复最近一次会话，或 claude --resume 选择历史会话。

新电脑恢复流程
bash
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
当前成果边界（2026-06-17）
已实现：

✅ Shell 配置恢复（.zshrc）

✅ Git 配置恢复（.gitconfig）

✅ VSCode 配置恢复（settings.json）

✅ Homebrew 软件清单恢复（Brewfile）

✅ API Key 与 SSH Key 安全隔离

✅ 新电脑一条命令恢复基础工作站配置

尚未实现（阶段2以后）：

❌ Python / Node / Java 版本自动安装

❌ Claude Code 自动配置（npm install -g @anthropic-ai/claude-code 及 DeepSeek endpoint 设置）

❌ Ollama 模型自动拉取（ollama pull）

❌ VSCode 扩展自动安装

当前完成的是 Workstation as Code Level 1——Dotfiles as Code，而不是 Full Bootstrap。工作站配置已固化，开发环境自动引导（Bootstrap）和项目模板化（Project Scaffolding）属于后续阶段。

当前状态快照（2026-06-17）
项目	状态
Git 仓库	git@github.com:l00-ai-workstation/Auto-WorkStation.git
最新 commit	以 GitHub main 分支最新提交为准
chezmoi	v2.70.5
Python	3.11.9 (pyenv)
Node	v24.16.0 (nvm)
Java	Temurin 21.0.11 (SDKMAN)
VSCode	执行 code --version 后填入实际值
Claude Code	v2.1.175，已配置 DeepSeek Anthropic Endpoint
Ollama（ollama list 输出）	deepseek-r1:8b / qwen3-vl:8b / qwen2.5:7b
API Key	~/.api_keys，权限 600，永不入仓
SSH Key	ed25519，已添加到 GitHub
待完成阶段
阶段2：AI Project Template（下一优先级）
背景：换电脑 1~3 年一次，新建项目每周甚至每天。项目模板的 ROI 远高于继续优化工作站恢复流程。

目标：新项目自动带 .venv / Python版本 / Node版本 / Java版本 / VSCode配置 / Claude Code 项目规则。

技术路线待定：A. Copier　B. Cookiecutter　C. 自定义脚本。当前倾向 Copier。

完成标志：执行 copier copy gh:l00-ai-workstation/ai-project-template myapp 一条命令即可生成包含完整运行时配置的新项目。

阶段3：Bootstrap Automation
目标：新电脑执行 chezmoi init --apply 后，通过 run_once 脚本自动完成以下所有安装，用户仅需补充 SSH Key 和 API Key 即可进入开发状态：

pyenv 安装 Python 3.11.9

nvm 安装 Node LTS

SDKMAN 安装 Java 21

Claude Code 安装及 DeepSeek endpoint 配置

Ollama 安装及模型拉取

VSCode 扩展安装

阶段4：Copier 工作流完善
目标：在阶段2基础上引入交互式项目生成，支持 project_name / python_version / node_version 等参数自定义，进一步打磨模板细节。

Document History
Version	Date	Notes
v1.0	2026-06-17	Phase 1 documentation finalized
Phase 1 至此完成并冻结。
除事实性错误修正外，本文档不再记录新的功能变更。
后续所有新增能力、自动化流程及架构演进统一记录于 Phase 2 及后续文档。