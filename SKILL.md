---
name: notion-sync
description: |
  Bidirectional sync between local markdown files and Notion pages.
  Triggers: user asks to sync, pull, push, or check sync status of documents;
  user asks AI to edit a document that has a Notion counterpart;
  user mentions sync-manifest.json or Notion sync.
  Workflow: register local-notion pairs in a manifest, pull before editing,
  push after editing, detect conflicts, heartbeat patrol.
version: 1.0.0
---

# Notion Sync: 本地 Markdown 与 Notion 双向同步

管理本地 .md 文件和 Notion 页面之间的双向同步。核心工具是项目根目录的 `sync-manifest.json`，记录每个文件的映射关系和同步状态。

## Manifest 格式

文件位置：项目根目录的 `sync-manifest.json`（与 AGENTS.md 同级）。

```json
{
  "project": "项目名称",
  "entries": [
    {
      "local_path": "drafts/paper.md",
      "notion_page_id": "a1b2c3d4e5f678901234567890abcdef",
      "title": "文档标题",
      "status": "synced",
      "last_synced": "2026-06-21T10:00:00+08:00",
      "local_hash": "sha256:abc123...",
      "notion_hash": "sha256:def456..."
    }
  ]
}
```

状态值：`synced`（两边一致）、`local-ahead`（本地有改动）、`remote-ahead`（Notion 有改动）、`conflict`（两边都改了）、`pending-init`（已注册但未计算 hash，需首次初始化）。

`local_path` 相对于 manifest 所在目录。

## 核心操作

### Init（首次初始化，针对 pending-init 条目）

1. 读 manifest，找到 `pending-init` 条目
2. 计算本地文件 hash
3. 用 `notion-fetch` 获取 Notion 内容，计算 hash
4. 更新 manifest：填入 `local_hash`、`notion_hash`、`last_synced`（当前时间）
5. 如果两个 hash 相同，`status = "synced"`；如果不同，`status = "conflict"`（交给用户决定）

### Pull（从 Notion 拉取到本地）

1. 读 manifest，定位目标条目
2. 用 `notion-fetch` 获取页面内容（传 `notion_page_id`），返回 Notion Markdown
3. 将内容写入 `local_path`
4. 计算 hash：本地文件 hash 和 Notion 内容 hash（此时相同）
5. 更新 manifest：`status = "synced"`，刷新 `local_hash`、`notion_hash`、`last_synced`

### Push（从本地推送到 Notion）

1. 读 manifest，定位目标条目
2. 读本地文件，计算 hash
3. 用 `notion-fetch` 获取 Notion 当前内容，计算 hash，与 manifest 中的 `notion_hash` 对比
4. 如果 Notion 端也变了（hash 不匹配），标记 `conflict` 并进入冲突处理，**不要直接覆盖**
5. 用 `notion-update-page` 推送：`command = "replace_content"`，`new_str` 为本地文件内容
6. 更新 manifest：`status = "synced"`，刷新两个 hash 和 `last_synced`

### Status（检查同步状态）

遍历 manifest 每个条目：

1. 计算本地文件当前 hash，与 manifest 中的 `local_hash` 对比
2. 用 `notion-fetch` 获取 Notion 内容，计算 hash，与 manifest 中的 `notion_hash` 对比
3. 判定状态：
   - 两者都没变 → `synced`
   - 仅本地变了 → `local-ahead`
   - 仅 Notion 变了 → `remote-ahead`
   - 都变了 → `conflict`
4. 更新 manifest 中的 status 字段
5. 输出摘要表格给用户

单个文件检查：只需检查指定文件，不需要遍历全部。

### Edit Workflow（编辑工作流，最重要）

用户要求编辑某个文档时，按以下步骤执行：

1. **查 manifest**：找到对应条目。如果文件不在 manifest 中，跳过同步直接编辑。
2. **检查状态**：如果 `pending-init`，先执行 Init 初始化。然后对该文件执行单文件 Status 检查。
3. **如果是 `remote-ahead`**：先 Pull，把 Notion 最新版本拉到本地
4. **如果是 `conflict`**：告知用户存在冲突，展示差异，等用户决定保留哪个版本
5. **执行编辑**：在本地 md 文件上操作
6. **Push 回 Notion**：编辑完成后推送
7. **更新 manifest**

如果用户说"基于 Notion 版本改"或类似表达，先 Pull 再编辑。

### Register（注册新文件）

用户提供本地文件路径和 Notion 页面 ID/URL 时：

1. 如果提供了 URL，从中提取 page ID（URL 中最后一段，32 个十六进制字符，去掉破折号）
2. 用 `notion-fetch` 验证页面存在
3. 计算本地文件 hash 和 Notion 内容 hash
4. 添加条目到 manifest，`status = "synced"`
5. 如果 manifest 不存在，先创建

### 从 Notion 创建本地文件

用户给出 Notion 页面想保存为本地 md 时：

1. `notion-fetch` 获取内容
2. 保存到用户指定的本地路径
3. Register 到 manifest

## Hash 计算

```bash
# 本地文件
shasum -a 256 <file> | cut -d' ' -f1

# Notion 内容（从 notion-fetch 返回的 Markdown 文本）
echo -n "<content>" | shasum -a 256 | cut -d' ' -f1
```

存储格式：`sha256:<前16位>` 以节省空间。

## 冲突处理

发现 `conflict` 时：

1. 展示本地版本和 Notion 版本的关键差异（不需要全文对比，挑主要改动）
2. 给用户三个选项：
   - **保留本地版本**：Push 本地内容覆盖 Notion
   - **保留 Notion 版本**：Pull Notion 内容覆盖本地
   - **智能合并**：AI 尝试合并两边的改动（需要人工确认）
3. 用户选择后执行对应操作，更新 manifest

## 注意事项

- Notion Markdown 支持大部分标准语法（标题、列表、代码块、表格、链接、图片等），但 Notion 特有元素（彩色文字、column layout、toggle block、synced block）在 pull-push 循环中可能丢失
- 评论和讨论不会被 fetch 回来，也不会被 push 上去
- 子页面和子数据库不会被同步，只同步页面本身的文本内容
- `notion-update-page` 的 `replace_content` 会检查是否删除了子页面/数据库，如果有会报错而非静默删除
- 如果 manifest 文件不存在，提示用户是否要创建一个

## 用户集成

### AGENTS.md 约定

用户应在项目的 AGENTS.md 中加入以下内容，确保 AI 每次编辑时自动检查同步状态：

```markdown
## Notion Sync

本项目使用 `sync-manifest.json` 管理本地 md 文件与 Notion 页面的双向同步。
- 编辑任何 .md 文件之前，先读取 sync-manifest.json（如存在）
- 如果目标文件在 manifest 中且状态为 remote-ahead 或 conflict，先同步再编辑
- 编辑完成后，将改动推送到 Notion 并更新 manifest
- 遵循 notion-sync skill 中定义的完整流程
```

### HEARTBEAT.md 巡检

用户可在 HEARTBEAT.md 中加入巡检任务：

```markdown
## Notion Sync Patrol

检查当前项目的 sync-manifest.json：
- 列出所有非 synced 状态的条目
- 如有 conflict，标记为紧急
- 如有文件超过 7 天未同步且状态非 synced，发出提醒
```
