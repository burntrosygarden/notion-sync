[English](README.md) | **中文**

# notion-sync

本地 Markdown 文件与 Notion 页面之间的双向同步。设计为 AI agent skill，让 AI 助手（QoderWork、Claude Code、Codex 等）自动保持本地文档和 Notion 页面的一致性。

## 为什么需要它

如果你在本地写 Markdown 然后同步到 Notion，两边的内容很快就会分叉。你在 Notion 上改了，下次让 AI 改本地文件时，它不知道 Notion 有更新的版本，直接覆盖了你的改动。反过来也一样。

`notion-sync` 用一个轻量的清单文件（`sync-manifest.json`）解决这个问题——记录哪些本地文件对应哪些 Notion 页面，以及当前的同步状态。AI 在每次编辑前读取这个清单，需要时先拉最新版本，编辑完再推回去。类似于 git status，但用于本地 ↔ Notion 的同步。

## 工作原理

### 三层架构

```
┌─────────────────────────────────────────────────┐
│  AGENTS.md          (每次会话自动加载)            │
│  → "编辑前必须检查 sync-manifest.json"           │
├─────────────────────────────────────────────────┤
│  notion-sync skill  (具体怎么做)                 │
│  → 拉取、推送、状态检查、冲突处理                 │
├─────────────────────────────────────────────────┤
│  HEARTBEAT.md       (定期巡检)                   │
│  → 发现过期/冲突的文件并提醒                      │
└─────────────────────────────────────────────────┘
```

- **AGENTS.md** 是触发器。它是项目级的约定文件，AI 在每次会话开始时读取。Notion Sync 部分告诉 AI：编辑任何 `.md` 文件之前，先检查 manifest。
- **Skill**（`SKILL.md`）包含具体的操作步骤——怎么从 Notion 拉取、怎么推送、怎么算 hash、怎么检测变更、怎么处理冲突。
- **HEARTBEAT.md** 是兜底。定期扫描，把漏掉的文件捞出来。

### 同步状态

manifest 中每个文件有一个状态：

| 状态 | 含义 |
|------|------|
| `synced` | 本地和 Notion 一致（hash 匹配） |
| `local-ahead` | 本地有改动，Notion 还没更新 |
| `remote-ahead` | Notion 有改动，本地还是旧版 |
| `conflict` | 两边都改了——需要人工决定保留哪个 |
| `pending-init` | 已注册但还没计算 hash（首次注册状态） |

### 编辑工作流

```
用户："帮我改一下 paper-draft.md"
  │
  ▼
AI 读取 sync-manifest.json
  │
  ├── 文件不在 manifest → 直接编辑，不走同步
  │
  ├── 状态: pending-init → 执行初始化（计算 hash）
  │
  ├── 状态: synced → 直接编辑
  │
  ├── 状态: local-ahead → 直接编辑
  │
  ├── 状态: remote-ahead → 先从 Notion 拉取最新版，再编辑
  │
  └── 状态: conflict → 展示差异，问用户保留哪个版本
  │
  ▼
AI 编辑本地 .md 文件
  │
  ▼
AI 推送到 Notion，更新 manifest → synced
```

## 项目配置

### 1. 创建 `sync-manifest.json`

放在项目根目录。完整格式见 `sync-manifest.example.json`。每个条目映射一个本地文件到一个 Notion 页面：

```json
{
  "project": "my-research",
  "entries": [
    {
      "local_path": "drafts/paper.md",
      "notion_page_id": "a1b2c3d4e5f678901234567890abcdef",
      "title": "My Paper",
      "status": "pending-init",
      "last_synced": null,
      "local_hash": null,
      "notion_hash": null
    }
  ]
}
```

`notion_page_id` 是 Notion 页面 URL 中的 32 位 UUID。例如 `https://notion.so/My-Page-a1b2c3d4e5f678901234567890abcdef` 中，ID 是 `a1b2c3d4e5f678901234567890abcdef`（去掉破折号）。

初始状态填 `"pending-init"` 和 null hash。skill 会在第一次使用时自动初始化。

### 2. 添加到 `AGENTS.md`

在项目的 `AGENTS.md` 末尾追加：

```markdown
## Notion Sync

本项目使用 `sync-manifest.json` 管理本地 md 文件与 Notion 页面的双向同步。
- 编辑任何 .md 文件之前，先读取 `sync-manifest.json`（如存在）
- 如果目标文件在 manifest 中且状态为 `remote-ahead` 或 `conflict`，先同步再编辑
- 编辑完成后，将改动推送到 Notion 并更新 manifest
- 遵循 `notion-sync` skill 中定义的完整流程
```

### 3.（可选）添加到 `HEARTBEAT.md`

定期巡检，在项目 `HEARTBEAT.md` 中加入：

```markdown
## Notion Sync 巡检

检查当前项目的 sync-manifest.json：
- 列出所有非 synced 状态的条目
- 如有 conflict，标记为紧急
- 如有文件超过 7 天未同步，发出提醒
```

## 完整示例

假设你有一篇论文草稿同时存在本地和 Notion 上。

**第一步：注册文件**

在 `sync-manifest.json` 中添加条目：

```json
{
  "local_path": "writing/paper-draft.md",
  "notion_page_id": "38584ccbacee814f873ef0a29e0f32a8",
  "title": "Paper Draft v3",
  "status": "pending-init",
  "last_synced": null,
  "local_hash": null,
  "notion_hash": null
}
```

**第二步：首次编辑**

你让 AI 改 `paper-draft.md`。AI 看到 `pending-init`，执行初始化：

1. 计算本地文件的 SHA-256 hash
2. 拉取 Notion 页面内容，计算 hash
3. 存入 manifest
4. hash 相同 → `synced`；不同 → `conflict`（问你保留哪个）

**第三步：正常编辑循环**

后来你在 Notion 上改了论文。下次让 AI 改本地文件：

1. AI 检查状态 → `remote-ahead`（Notion hash 变了）
2. AI 拉取 Notion 版本，覆盖本地文件
3. AI 执行你的编辑指令
4. AI 推回 Notion
5. 更新 manifest → `synced`

**第四步：冲突处理**

你在本地和 Notion 上都改了，没来得及同步。AI 发现 `conflict`：

1. AI 展示两个版本的主要差异
2. 你选择：保留本地、保留 Notion、或让 AI 智能合并
3. AI 执行并推送最终版本

## 依赖

- **Notion MCP 集成** — AI 需要通过 Notion MCP server 访问 Notion 页面
- **`shasum`** — 用于 hash 计算（macOS / Linux 自带）
- **支持读取 AGENTS.md 的 AI agent** — QoderWork、Claude Code、Codex 等

## 文件结构

```
your-project/
├── AGENTS.md                 ← 包含 Notion Sync 约定
├── HEARTBEAT.md              ← 可选的巡检任务
├── sync-manifest.json        ← 同步清单
└── writing/
    └── paper-draft.md        ← 你的文档
```

## License

MIT
