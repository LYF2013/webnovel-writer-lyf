# 本地开发者模式 · 安装与使用指南

> 适用对象：把本仓库当源代码维护、希望「git pull / 自己改码」就立刻在 Claude Code 中生效的开发者。
> 与 README 的「快速开始」面向普通用户不同，本文档强调让 Claude Code 直接运行你**本地仓库**的代码，而不是 Marketplace 缓存目录中的发布版本。

---

## 一、核心原理（必读）

Claude Code 运行插件时读取的是环境变量 `CLAUDE_PLUGIN_ROOT` 指向的目录，**不是**你的 git 仓库目录。所以单纯 `git pull` 不会让 Claude 用上新代码。

本地开发模式的本质是：**把 `CLAUDE_PLUGIN_ROOT` 钉到你本地仓库的 `webnovel-writer/` 子目录**，从此修改 → 新会话 → 立即生效。

仓库关键路径：

- `webnovel-writer/.claude-plugin/plugin.json`：插件元数据
- `.claude-plugin/marketplace.json`：marketplace 元数据（仓库根）
- `webnovel-writer/`：插件源码主体

---

## 二、一次性设置步骤

### 1) 卸载已存在的远程版本（避免与本地版冲突）

```bash
claude plugin uninstall webnovel-writer --scope user
claude plugin marketplace remove lingfengQAQ/webnovel-writer
```

> 之前装的若是 `--scope project`，对应改成 project。如果从未安装过远程版本可跳过本步。

### 2) 把本地仓库注册为 Marketplace 源

```bash
claude plugin marketplace add /Users/liyufei/Documents/AIApplication/webnovel-writer-lyf
```

`marketplace add` 后面跟的是**仓库根目录绝对路径**（即包含 `.claude-plugin/marketplace.json` 的那一层）。

### 3) 从本地 Marketplace 安装插件

```bash
claude plugin install webnovel-writer@webnovel-writer-marketplace --scope user
```

安装完成后，Claude Code 的 `CLAUDE_PLUGIN_ROOT` 会指向 `/Users/liyufei/Documents/AIApplication/webnovel-writer-lyf/webnovel-writer/`。

### 4) 安装 Python 依赖

推荐使用项目自带虚拟环境：

```bash
source .venv/bin/activate
python -m pip install -r requirements.txt
```

如不使用虚拟环境，确保用与 Claude Code 相同的 Python 解释器执行 pip。

### 5) 验证主链健康

在 Claude Code 中执行：

```text
/webnovel-preflight
```

输出 `story_runtime: healthy` 即代表本地版插件已被 Claude 正确加载。

---

## 三、日常开发循环

### 改了什么 → 是否需要重装

| 修改对象 | 是否重装 | 操作 |
|---|---|---|
| `webnovel-writer/scripts/` 下的 Python 脚本 | 否 | 新开会话即可 |
| `webnovel-writer/skills/*.md`（命令定义） | 否 | 新开会话即可 |
| `webnovel-writer/agents/*.md`（agent prompt） | 否 | 新开会话即可 |
| `webnovel-writer/templates/`、`webnovel-writer/references/` | 否 | 即时生效 |
| `webnovel-writer/.claude-plugin/plugin.json` | 是 | 重新 install |
| `.claude-plugin/marketplace.json` | 是 | marketplace update + install |
| `requirements.txt` | 是 | 重新 pip install |
| `webnovel-writer/dashboard/frontend/` 源码 | 是 | npm install && npm run build |

### 元数据变更后的重装命令

```bash
claude plugin marketplace update webnovel-writer-marketplace
claude plugin install webnovel-writer@webnovel-writer-marketplace --scope user
```

### Dashboard 前端改动后的构建命令

```bash
cd webnovel-writer/dashboard/frontend
npm install
npm run build
```

后端 `dashboard/app.py` 修改不需要构建，重启 dashboard 进程即可。

---

## 四、与 README「快速开始」的差异速查

| 步骤 | README 模式（普通用户） | 本地开发模式 |
|---|---|---|
| 1. 安装插件 | `marketplace add lingfengQAQ/webnovel-writer`（拉远程） | `marketplace add /Users/liyufei/.../webnovel-writer-lyf`（指本地） |
| 2. 安装依赖 | `pip install -r https://raw.githubusercontent.com/.../requirements.txt` | `pip install -r requirements.txt`（本地） |
| 3. /webnovel-init | 完全相同 | 完全相同 |
| 4. 配 .env | 完全相同 | 完全相同 |
| 5. 写作命令 | 跑稳定发布版 | 跑你 HEAD 上的代码，改完新会话即生效 |
| 6. Dashboard | 用预构建产物 | 改前端需自行 `npm run build` |
| 7. agents 模型 | 完全相同 | 完全相同 |

一句话总结：**README 模式跑的是 GitHub 已发布版；本地开发模式把"插件目录"指针钉到你本地仓库，从此修改即生效，不再走"发版-重装"循环。**

---

## 五、开发者自检命令速查

| 目的 | 命令 |
|---|---|
| 查看当前插件指向（验证是否为本地路径） | `claude plugin list` |
| 主链健康检查 | `/webnovel-preflight` |
| 跑测试 | `pytest`（配置见 `pytest.ini`） |
| 三处版本一致性同步 | `python -X utf8 webnovel-writer/scripts/sync_plugin_version.py --version X.Y.Z --release-notes "..."` |

三处版本一致性指：

- `webnovel-writer/.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `README.md`

CI 工作流 `.github/workflows/plugin-version.yml` 会在 PR 时强校验。

---

## 六、避坑清单

1. **Claude 读的是 `CLAUDE_PLUGIN_ROOT`，不是 git 仓库目录**。如果跳过 marketplace add / install 直接 git pull，Claude 仍在跑旧缓存，常见误以为「改了没生效」的根因。
2. **`.story-system/` 是真源**，不要手动改 `.webnovel/state.json`、`index.db`、`summaries/` 等投影产物。调试时投影坏了，删掉 `.webnovel/*` 由 projection writers 重建。
3. **scope 要统一**：全流程 `--scope user` 或全流程 `--scope project`，不要混用。
4. **新开会话最稳**：修改 skill / agent 的 md 描述后，已开会话可能缓存旧 prompt，新开会话保险。
5. **版本号三处必须一致**：改任一处都用 `sync_plugin_version.py` 一把梭，避免 CI 红。

---

## 七、推荐 Git 工作流

```bash
git checkout -b feature/your-feature
# 写码 + 本地 Claude Code 验证
pytest
git commit -m "feat: ..."
git push origin feature/your-feature
# 在 GitHub 开 PR，CI 自动跑 plugin-version 与 plugin-release 校验
```

---

## 八、卸载本地安装

当你想停止使用本地开发模式（例如改回 Marketplace 版、彻底清理环境、或换一个仓库路径重新挂载）时，按以下步骤操作。

### 1) 卸载本地安装的插件

```bash
claude plugin uninstall webnovel-writer --scope user
```

> 当初装的若是 `--scope project`，对应改成 project。

### 2) 移除本地 Marketplace 注册

```bash
claude plugin marketplace remove webnovel-writer-marketplace
```

> 这里的名字是仓库根 `.claude-plugin/marketplace.json` 中的 `name` 字段，不是仓库路径。如果忘了，可以先用 `claude plugin marketplace list` 查看当前注册的所有 marketplace。

### 3) 验证卸载结果

```bash
claude plugin list
claude plugin marketplace list
```

输出中应不再出现 `webnovel-writer` 与 `webnovel-writer-marketplace`。

### 4) 可选：清理 Python 依赖

如果该项目的依赖只为本插件服务，且不再需要：

```bash
# 推荐：直接删除项目内的虚拟环境
rm -rf .venv

# 或者在已激活的 venv 中卸载本项目依赖
python -m pip uninstall -r requirements.txt -y
```

### 5) 可选：清理书项目数据

插件本身不存数据，所有写作产物都在你的「书项目目录」（如 `~/novels/xxx/`）。卸载插件**不会**删除任何书项目的内容。如果某本书也确认不再用：

```bash
rm -rf ~/novels/<your-book-dir>
```

> 谨慎：删除前请确认 `.story-system/` 没有未备份的真源数据。建议先按 `docs/operations/operations.md` 的备份指引归档。

### 6) 可选：恢复使用 Marketplace 版本

卸载本地版后，如需切回官方发布版：

```bash
claude plugin marketplace add lingfengQAQ/webnovel-writer --scope user
claude plugin install webnovel-writer@webnovel-writer-marketplace --scope user
```

### 7) 卸载常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `uninstall` 报「插件不存在」 | scope 不对 | 用 `claude plugin list --scope project` / `--scope user` 查实际所在 scope，再用对应 scope 卸载 |
| `marketplace remove` 报「找不到 marketplace」 | 名字写错 | `claude plugin marketplace list` 复制准确名字再删 |
| 卸载后 `/webnovel-*` 命令仍出现在 Claude 会话 | 旧会话缓存了 skill 列表 | 新开 Claude Code 会话即可 |
| 想保留代码但暂时停用 | 不必卸载 | 直接 `claude plugin uninstall` 就够；marketplace 注册可保留，下次只需重新 install |

---

## 九、相关文档

- `docs/architecture/overview.md`：系统架构与 Story System 主链
- `docs/operations/operations.md`：目录口径、备份、恢复
- `docs/operations/plugin-release.md`：发版流程
- `docs/guides/commands.md`：所有命令详解
- `docs/guides/rag-and-config.md`：RAG 与 .env 配置