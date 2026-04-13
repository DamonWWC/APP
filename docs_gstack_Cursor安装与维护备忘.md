# gstack · Cursor 安装与维护备忘

本文记录 **本机已采用的安装方式** 与 **日常维护注意事项**，便于日后查阅。通用概念与完整技能说明见同目录下的 [`docs_gstack_Cursor使用指南.md`](./docs_gstack_Cursor使用指南.md)。

---

## 1. 本机路径约定

| 项目 | 路径 |
|------|------|
| gstack 源码仓库 | `E:\Github\gstack` |
| Cursor 用户级技能目录 | `C:\Users\Admin\.cursor\skills\`（即 `%USERPROFILE%\.cursor\skills\`） |
| 仓库内生成的 Cursor 技能 | `E:\Github\gstack\.cursor\skills\gstack-*` |

**重要：** `C:\Users\Admin\.cursor\skills` 下的 **`gstack*`** 目录多为 **目录联接（Junction）**，指向 `E:\Github\gstack\.cursor\skills\...`；运行时根目录 `gstack` 内的 `bin`、`browse\dist` 等也联接至仓库。**请勿随意移动、重命名或删除 `E:\Github\gstack`**，否则 Cursor 侧技能会失效。

---

## 2. 为何没用 `./setup --host cursor`

当前仓库版本中的 `setup` 脚本在 `case "$HOST"` 里 **尚未接受 `cursor`**，执行 `./setup --host cursor` 会报 `Unknown --host value`。

本机采用 **等效手动流程**（已与官方 Codex/外部 Host 思路一致）：

1. 在仓库根目录执行：`bun install`、`bun run gen:skill-docs --host cursor`、`bun run build`。
2. 在 `%USERPROFILE%\.cursor\skills` 下：
   - 创建 **`gstack`** 运行时目录：复制根技能 `SKILL.md`，对 `bin`、`browse\dist`、`browse\bin`、`design\dist` 做 **Junction** 指向仓库；复制 `ETHOS.md`、`review` 下若干清单文件及 `gstack-upgrade\SKILL.md`。
   - 对其余 **`gstack-*`** 技能文件夹，用 **Junction** 指向 `E:\Github\gstack\.cursor\skills\` 下同名目录。

若未来官方 `setup` 已支持 `--host cursor`，可考虑改为官方一键安装；迁移前请先备份或删除现有 `~\.cursor\skills\gstack*` 再按官方说明操作。

---

## 3. 安装后必做

- **完全退出并重新打开 Cursor**（或新开 Agent 会话），以便重新加载 Skills。
- 确认本机已安装 **Bun**，且 **Windows 上建议同时安装 Node.js**（Playwright / 浏览器相关能力官方说明会回退到 Node）。

---

## 4. 技能命名与调用

- 当前生成结果为 **带前缀** 的目录名：`gstack-office-hours`、`gstack-review`、`gstack-qa` 等（与 `gen:skill-docs --host cursor` 输出一致）。
- 在 Cursor 中请以实际加载的 Skill 名为准；若界面支持斜杠命令，形式取决于 Cursor 版本与 Skills 展示方式。

---

## 5. 日常更新 gstack

在 **`E:\Github\gstack`** 下执行：

```powershell
cd E:\Github\gstack
git pull
bun install
bun run gen:skill-docs --host cursor
bun run build
```

说明：

- **`gen:skill-docs --host cursor`** 会更新仓库内 `.cursor\skills\` 下的 `SKILL.md`；因用户目录通过 **Junction** 指向此处，**一般无需重新做联接**。
- **`bun run build`** 会重新编译 browse、design 等二进制；若仅改了文档、未改 native 部分，可按需执行；大版本或官方 Changelog 要求时再跑全量 build 更稳妥。

---

## 6. 重装 / 换机 / 更换仓库路径

若需 **彻底清除 Cursor 侧 gstack**：

1. 关闭 Cursor。
2. 删除 `%USERPROFILE%\.cursor\skills` 下所有以 **`gstack`** 开头的文件夹（含 `gstack` 与 `gstack-*`）。
3. 在新路径克隆仓库或恢复 `E:\Github\gstack` 后，重复第二节中的构建与 Junction 步骤（或改用未来官方支持的 `./setup --host cursor`）。

**更换盘符或目录** 后，必须 **重新创建 Junction**，旧联接不会自动跟随。

---

## 7. 故障速查

| 现象 | 处理方向 |
|------|-----------|
| 技能不出现或未更新 | 重启 Cursor；确认 `E:\Github\gstack` 仍存在且 `git pull` + `gen:skill-docs --host cursor` 已执行。 |
| `/browse`、浏览器类失败 | 在仓库执行 `bun install` 与 `bun run build`；Windows 确认 `node` 在 PATH 且可加载 `playwright`。 |
| 提示找不到 `bin` 或脚本 | 检查 `C:\Users\Admin\.cursor\skills\gstack\bin` 是否为有效 Junction 且目标为 `E:\Github\gstack\bin`。 |
| 误删仓库 | 从 GitHub 重新 `git clone` 到原路径或按第六节重做联接。 |

---

## 8. 相关链接

- 上游仓库：<https://github.com/garrytan/gstack>
- 本仓库内更完整的概念、技能表与官方路径说明：[`docs_gstack_Cursor使用指南.md`](./docs_gstack_Cursor使用指南.md)

---

*文档生成说明：基于本机已完成的 Cursor + gstack（Junction）安装方式整理；若你更改了用户名或仓库路径，请同步修改本文第一节表格。*
