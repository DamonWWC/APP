# gstack 使用指南（含 Cursor 安装）

本文档基于开源仓库 [garrytan/gstack](https://github.com/garrytan/gstack) 的 README、架构说明与 `hosts/cursor.ts` 整理，面向 **在 Cursor 中使用 gstack** 的场景。

---

## 一、gstack 是什么

**gstack** 是 YC CEO Garry Tan 开源的一套「AI 工程工作流」资源：**大量以 Markdown 编写的 Skill（技能）**，通过 **斜杠命令**（如 `/office-hours`、`/review`、`/qa`）驱动，把单一 AI 会话组织成类似「产品 / 设计 / 工程 / 评审 / QA / 发布」的分工流水线。

**特点简述：**

- **MIT 开源**，仓库内是技能说明、脚本与浏览器自动化相关代码（含 Playwright、自研 browse 工具链等）。
- 最初为 **Claude Code** 优化，现已通过 **多 Host 配置** 支持多种 AI 编程工具（含 Cursor、Codex CLI、OpenCode 等，详见仓库 `hosts/`）。
- 核心节奏：**Think → Plan → Build → Review → Test → Ship → Reflect**，各技能之间通过设计文档、测试计划等产物衔接。

**官方仓库：** [https://github.com/garrytan/gstack](https://github.com/garrytan/gstack)

---

## 二、安装前准备

### 2.1 通用依赖

| 依赖 | 说明 |
|------|------|
| **Git** | 克隆仓库 |
| **Bun** ≥ 1.0 | 运行 `setup`、构建 browse、生成各 Host 的 Skill 文档（`package.json` 中 `engines.bun`） |
| **Node.js** | **Windows 上强烈必需**：官方说明因 Bun 在 Windows 上与 Playwright 管道传输存在已知问题（[bun#4253](https://github.com/oven-sh/bun/issues/4253)），浏览相关能力会回退到 Node 启动 Chromium |

### 2.2 Windows 特别说明

- 官方建议在 **Windows 11** 下使用 **Git Bash** 或 **WSL** 执行 `./setup`（脚本为 Bash）。
- 请确保 `bun` 与 `node` 均在 `PATH` 中。

### 2.3 Cursor 侧 Skill 目录（官方 Host 配置）

根据仓库中的 [`hosts/cursor.ts`](https://github.com/garrytan/gstack/blob/main/hosts/cursor.ts)：

- 全局技能根路径为 **`~/.cursor/skills/gstack`**（生成物通常为多个 `gstack-*` 技能目录，README 中写作 `~/.cursor/skills/gstack-*/`）。
- Windows 下一般为：`C:\Users\<你的用户名>\.cursor\skills\`。

安装完成后，**重启 Cursor 或新开对话**，以便加载 Skills。

---

## 三、在 Cursor 上安装 gstack（推荐流程）

> **说明：** 官方 README 写明可使用 `./setup --host cursor`。若你克隆到的版本中 `setup` 报错 `Unknown --host value: cursor`，说明该版本 Bash 脚本尚未把 `cursor` 列入合法 `--host`（仓库演进中 README 与脚本可能短暂不同步）。此时请使用 **第三节「备用：仅生成 Cursor 技能文档」**。

### 3.1 克隆仓库

浅克隆即可（官方 Quick start 推荐）：

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/gstack
```

Windows（Git Bash）示例：

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git "$HOME/gstack"
cd "$HOME/gstack"
```

### 3.2 执行 setup（指定 Cursor）

```bash
cd ~/gstack
chmod +x setup
./setup --host cursor
```

**常用参数：**

| 参数 | 作用 |
|------|------|
| `--host cursor` | 仅为 Cursor 安装 / 链接技能 |
| `-q` / `--quiet` | 安静模式；非交互时技能命名默认倾向短命令（见脚本内逻辑） |
| `--no-prefix` | 使用 `/qa`、`/ship` 等短名（与 `--prefix` / `gstack-` 前缀互斥，偏好会写入配置） |
| `--prefix` | 使用 `/gstack-qa` 等带前缀名称，避免与其它技能包冲突 |

### 3.3 安装后自检

1. 检查目录是否存在类似：`%USERPROFILE%\.cursor\skills\gstack-*`（或仓库/文档中说明的符号链接目标）。
2. 在 Cursor Agent 对话中尝试输入 **`/office-hours`** 或 **`/review`**（具体是否显示为「斜杠菜单」取决于 Cursor 版本与 Skills 加载方式；若未出现，可在对话中直接写「请按 gstack 的 office-hours 流程执行」并 @ 相关 Skill 文件）。
3. 若使用浏览器类技能失败，在仓库根目录执行：

   ```bash
   bun install
   bun run build
   ```

   然后重新执行 `./setup --host cursor`。

### 3.4 备用：仅生成 Cursor 技能文档（当 `./setup --host cursor` 不可用）

在仓库根目录：

```bash
bun install
bun run gen:skill-docs --host cursor
```

生成物会落在仓库内由 Host 配置决定的目录（与 `hosts/cursor.ts` 中 `globalRoot` / 生成脚本一致，一般为 **`.cursor/skills/`** 下的 `gstack-*`）。你需要将这些技能目录 **复制或符号链接** 到本机 Cursor 的用户级 skills 目录 **`%USERPROFILE%\.cursor\skills\`**（与官方 README 表格一致），然后重启 Cursor。

> 详细 Host 扩展方式见：[docs/ADDING_A_HOST.md](https://github.com/garrytan/gstack/blob/main/docs/ADDING_A_HOST.md)

---

## 四、让项目里的 AI「记得用 gstack」

gstack 在 Claude Code 场景下常通过 **`CLAUDE.md`** 中的一段「gstack」说明，列出可用斜杠命令与浏览策略。在 **Cursor** 中建议等价做法：

1. 在项目根目录维护 **`AGENTS.md`** 或 **`.cursor/rules`** 中的规则，写明：
   - 进行 Web 浏览 / 验收时优先遵循 gstack 的 **`/browse`** 等技能说明；
   - 列出你本机已安装的技能名称（与 `~/.cursor/skills` 下目录一致）。
2. 若团队使用 Claude Code + Team 模式，仓库里可能有 `.claude/` 与 `CLAUDE.md` 的约定；Cursor 用户可只摘取其中 **技能列表与流程** 到本项目的 `AGENTS.md`，避免工具名混用。

README 中给 Claude 的 gstack 段落示例（需把路径与工具名改为 Cursor 语境）：

- 使用 gstack 的 **`/browse`** 做网页浏览；
- 避免与仓库中声明冲突的 MCP 工具混用（按你实际安装的 MCP 调整）；
- 列出：`/office-hours`、`/plan-ceo-review`、`/plan-eng-review`、`/plan-design-review`、`/design-consultation`、`/design-shotgun`、`/design-html`、`/review`、`/ship`、`/land-and-deploy`、`/canary`、`/benchmark`、`/browse`、`/open-gstack-browser`、`/qa`、`/qa-only`、`/design-review`、`/setup-browser-cookies`、`/setup-deploy`、`/retro`、`/investigate`、`/document-release`、`/codex`、`/cso`、`/autoplan`、`/plan-devex-review`、`/devex-review`、`/pair-agent`、`/careful`、`/freeze`、`/guard`、`/unfreeze`、`/gstack-upgrade`、`/learn` 等（以你安装版本为准）。

---

## 五、技能总览与使用场景

### 5.1 标准迭代流水线（建议顺序）

| 阶段 | 代表命令 | 角色定位（意译） |
|------|-----------|------------------|
| 想清楚 | `/office-hours` | 用多个强约束问题重构需求，输出设计向文档 |
| 战略与范围 | `/plan-ceo-review` | 从「创始人视角」挑战范围与方向 |
| 工程与架构 | `/plan-eng-review` | 数据流、边界情况、测试矩阵、风险 |
| 设计质量 | `/plan-design-review` | 设计维度打分、「AI slop」识别与改写计划 |
| 开发者体验 | `/plan-devex-review` | 面向 API/CLI/SDK/文档的 DX 规划 |
| 一站式规划 | `/autoplan` | 自动串联 CEO / 设计 / 工程 / DX 等评审（按适用性） |
| 实现后评审 | `/review` | 深度代码评审，部分问题可自动修 |
| 调试 | `/investigate` | 强调先调查再修复的根因分析 |
| 上线前测试 | `/qa <URL>` | 真实浏览器走查、修 bug、补回归测试 |
| 仅报告 | `/qa-only` | 只出报告不改代码 |
| 发布 | `/ship` | 同步分支、测、覆盖率审计、提 PR 等 |
| 合并与部署 | `/land-and-deploy` | 合并 PR、等 CI/部署并做生产健康检查 |
| 发布后 | `/canary`、`/benchmark` | 监控、性能基线对比 |
| 文档同步 | `/document-release` | 按本次发布更新 README/架构文档等 |
| 复盘 | `/retro` | 周度工程复盘（含 `/retro global` 跨项目） |

### 5.2 设计向子流程

- **`/design-consultation`**：从 0 搭设计系统、竞品与风险。
- **`/design-shotgun`**：多方案视觉探索 + 浏览器对比板 + 口味记忆。
- **`/design-html`**：把定稿视觉转为更可投产的 HTML/CSS（含 Pretext 等布局思路，见 README）。
- **`/design-review`**：对已实现界面做设计审计并直接改。

### 5.3 安全与「第二意见」

- **`/cso`**：OWASP Top 10 + STRIDE，强调低噪声、高置信度与可验证场景。
- **`/codex`**：调用 OpenAI **Codex CLI** 做独立评审或对抗式挑战（需本机已安装 Codex CLI；与 Claude/Cursor 侧 `/review` 可形成交叉模型对比）。

### 5.4 浏览器与多智能体

- **`/browse`**：无头/可控 Chromium，供 Agent「看见」页面。
- **`/open-gstack-browser`**：带侧边栏等能力的 GStack Browser（见 README 与 [BROWSER.md](https://github.com/garrytan/gstack/blob/main/BROWSER.md)）。
- **`/setup-browser-cookies`**：从本机 Chrome/Arc/Brave/Edge 导入 Cookie，测登录态页面。
- **`/pair-agent`**：与其它 Agent（OpenClaw、Codex、Cursor 等）共享浏览器会话，分 Tab、令牌隔离等。

### 5.5 安全护栏与编辑范围

- **`/careful`**：对 `rm -rf`、`DROP TABLE`、强推等破坏性操作预警。
- **`/freeze`**：限制编辑范围在指定目录。
- **`/guard`**：`/careful` + `/freeze`。
- **`/unfreeze`**：解除 `/freeze`。

### 5.6 其它工具向

- **`/gstack-upgrade`**：升级 gstack 并同步多位置安装。
- **`/learn`**：跨会话的「学习记忆」管理（项目偏好、坑点等）。
- **`/setup-deploy`**：为 `/land-and-deploy` 做一次性部署配置。

### 5.7 我该用哪个「评审」？

（与 README 表格一致，翻译为中文）

| 你在做… | 编码前（计划阶段） | 上线后（实况审计） |
|---------|-------------------|-------------------|
| 终端用户产品（UI / Web / App） | `/plan-design-review` | `/design-review` |
| 开发者产品（API / CLI / SDK / 文档） | `/plan-devex-review` | `/devex-review` |
| 架构与工程质量 | `/plan-eng-review` | `/review` |
| 以上全都要 | `/autoplan` | — |

---

## 六、与 Claude Code / OpenClaw / 其它 Host 的关系

- **Claude Code**：默认路径为 `~/.claude/skills/gstack`，可用 `./setup` 或 README 中的「粘贴给 Claude」一键说明。
- **Team 模式**：`./setup --team` 会在 Claude Code 设置里注册 SessionStart 钩子做自动更新；**主要针对 Claude Code**，Cursor 用户一般只需本机全局安装 + 自行 `git pull` / `/gstack-upgrade`。
- **OpenClaw**：`./setup --host openclaw` 会退出并提示改为方法论产物或阅读 [docs/OPENCLAW.md](https://github.com/garrytan/gstack/blob/main/docs/OPENCLAW.md)。
- **多 Host**：同一台机器可 `./setup --host auto` 检测多个 CLI；或分别 `--host codex`、`--host cursor` 等。

---

## 七、隐私与遥测

- **默认关闭**；首次运行可交互选择是否发送匿名使用统计。
- 若开启，上报字段限于：技能名、耗时、成功/失败、gstack 版本、操作系统等（**不含**代码、路径、仓库名、分支、提示词正文）。
- 可随时：`gstack-config set telemetry off`（需本机已安装 gstack 的 `bin` 工具链）。
- 本地可使用 **`gstack-analytics`** 查看本地 JSONL 统计（无需联网）。

---

## 八、故障排除

| 现象 | 建议 |
|------|------|
| 技能未出现 | 回到仓库目录执行 `./setup --host cursor`；确认 `~/.cursor/skills` 下存在 `gstack-*` |
| `/browse` 失败 | `bun install && bun run build` 后重装；Windows 确认 Node 可 `require('playwright')` |
| 安装过旧 | 仓库内 `git pull` 后执行 `/gstack-upgrade` 或再次 `./setup --host cursor` |
| `--host cursor` 报错 Unknown host | 使用 **第三节备用**：`bun run gen:skill-docs --host cursor` 并手动同步到 `%USERPROFILE%\.cursor\skills` |
| Codex 相关 | `/codex` 依赖 OpenAI Codex CLI；与 Cursor 独立 |

**卸载（概要）：** 仓库提供 `bin/gstack-uninstall`；并手动清理 `~/.cursor/skills` 下 gstack 相关链接/目录与 `~/.gstack`（详见 README「Uninstall」）。

---

## 九、延伸阅读（官方文档）

| 文档 | 内容 |
|------|------|
| [README.md](https://github.com/garrytan/gstack/blob/main/README.md) | 安装、技能表、并行冲刺、语音触发等 |
| [docs/skills.md](https://github.com/garrytan/gstack/blob/main/docs/skills.md) | 各技能深度说明与示例 |
| [ARCHITECTURE.md](https://github.com/garrytan/gstack/blob/main/ARCHITECTURE.md) | 系统架构 |
| [BROWSER.md](https://github.com/garrytan/gstack/blob/main/BROWSER.md) | `/browse` 命令参考 |
| [ETHOS.md](https://github.com/garrytan/gstack/blob/main/ETHOS.md) | 构建者方法论 |
| [CONTRIBUTING.md](https://github.com/garrytan/gstack/blob/main/CONTRIBUTING.md) | 开发者构建与测试 |

---

## 十、本次环境执行情况（供你本地继续）

在当前自动化环境中曾尝试：

1. 通过官方脚本安装 **Bun** → 访问 `github.com` 超时失败。
2. **`git clone https://github.com/garrytan/gstack.git`** → 连接 `github.com:443` 超时失败。

因此 **未能在你这台机器上完成克隆与 `./setup`**。请你在本机网络可访问 GitHub 时，在 **Git Bash** 中执行：

```bash
# 1）安装 Bun（任选其一，以官网为准）
# PowerShell: irm bun.sh/install.ps1 | iex

# 2）克隆并安装 Cursor 技能
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git "$HOME/gstack"
cd "$HOME/gstack"
./setup --host cursor -q
```

完成后重启 **Cursor**，并在项目里用 `AGENTS.md` 或 `.cursor/rules` 写明使用 gstack 的流程与技能列表。

---

*文档版本：与 gstack 仓库 main 分支公开内容对照整理；具体命令以你克隆时的仓库版本为准。*
