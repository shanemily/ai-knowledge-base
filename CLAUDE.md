# AGENTS.md

> 本文件是 AI 知识库助手项目的 Agent 操作手册。所有自动化 Agent（采集 / 分析 / 整理 / 分发）在执行任务前必须阅读本文件，并严格遵守其中的规范与红线。

## 1. 项目概述

本项目是一个面向 **AI / LLM / Agent 领域** 的自动化知识库助手：周期性地从 **GitHub Trending** 与 **Hacker News** 采集前沿技术动态，经由大模型完成清洗、摘要、打标签等结构化分析后，以 **JSON** 形式落库，并支持通过 **Telegram / 飞书** 等多渠道推送给订阅者。目标是让使用者每天用最少的时间获取经过筛选与提炼的高质量 AI 技术情报。

## 2. 技术栈

| 类别 | 选型 |
|---|---|
| 语言 | Python 3.12 |
| 智能体框架 | OpenCode + 国产大模型（如 DeepSeek / 通义 / Kimi 等，按 `.opencode/` 配置） |
| 编排引擎 | LangGraph（用于多 Agent 状态机与流程编排） |
| 工具/插件协议 | OpenClaw（统一封装外部工具调用） |
| 数据格式 | JSON（落库）、Markdown（人读视图） |
| 分发渠道 | Telegram Bot、飞书机器人 Webhook |

## 3. 编码规范

- 严格遵循 **PEP 8**（缩进 4 空格、单行 ≤ 99 字符、import 分组）。
- 命名一律使用 **snake_case**；类名 PascalCase；常量 UPPER_SNAKE_CASE。
- 所有公开函数、类、模块必须写 **Google 风格 docstring**（含 Args / Returns / Raises）。
- **禁止裸 `print()`** —— 一律使用 `logging` 模块（开发期 DEBUG，生产期 INFO 起步）。
- 类型注解必须完整：函数签名、返回值、容器内元素类型。
- 异常必须显式捕获具体类型，禁止 `except:` 与 `except Exception` 兜底（除非紧接 re-raise 或日志后再抛）。
- I/O、网络调用一律走超时与重试封装；外部 URL 必须校验 scheme。
- 提交前运行 `ruff check` / `ruff format` / `mypy`，全部通过方可入库。

## 4. 项目结构

```
ai-knowledge-base/
├── .opencode/
│   ├── agents/          # 各 Agent 的人格、职责与提示词定义
│   └── skills/          # 可复用的技能脚本（采集器、分析器、分发器等）
├── knowledge/
│   ├── raw/             # 原始抓取数据（按日期分目录的 JSON / HTML 快照）
│   └── articles/        # 经分析整理后的结构化知识条目（JSON）
├── README.md
└── AGENTS.md            # 本文件
```

> 注：仓库当前存在 `knowledge/artic/` 目录，应是 `articles/` 的截断写法，建议在首次提交前重命名以与本文档一致。

## 5. 知识条目 JSON 格式

每条知识条目为一个独立 JSON 文件，存放于 `knowledge/articles/<YYYY-MM-DD>/<id>.json`，结构如下：

```json
{
  "id": "20260503-gh-001",
  "title": "Awesome-LLM-Agents: 全面整理 2026 年最新 Agent 框架",
  "source": "github_trending",
  "source_url": "https://github.com/example/awesome-llm-agents",
  "author": "example",
  "published_at": "2026-05-03T08:21:00+08:00",
  "collected_at": "2026-05-03T09:00:00+08:00",
  "language": "zh",
  "summary": "一句话提要：……（≤ 80 字）",
  "highlights": [
    "亮点 1：……",
    "亮点 2：……"
  ],
  "tags": ["LLM", "Agent", "Framework", "Awesome-List"],
  "category": "tooling",
  "score": 87,
  "status": "published",
  "distributed_to": ["telegram", "feishu"],
  "raw_ref": "knowledge/raw/2026-05-03/gh-001.json"
}
```

字段约束：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 全局唯一，格式 `YYYYMMDD-<source>-<seq>` |
| `title` | string | 原标题（保留原文） |
| `source` | enum | `github_trending` / `hacker_news` |
| `source_url` | string (URL) | 必须为 https，且通过可达性校验 |
| `summary` | string | 中文一句话提要，≤ 80 字 |
| `tags` | string[] | 2–6 个，使用受控词表 |
| `score` | int | 0–100，由分析 Agent 给出的价值评分 |
| `status` | enum | `draft` / `reviewed` / `published` / `archived` |
| `distributed_to` | string[] | 已成功分发的渠道 |

## 6. Agent 角色概览

| 角色 | Agent 名 | 主要职责 | 输入 | 输出 |
|---|---|---|---|---|
| 采集 | `collector` | 从 GitHub Trending、Hacker News 拉取候选条目，做去重与基础清洗 | 数据源配置、上次水位线 | `knowledge/raw/<date>/*.json` |
| 分析 | `analyzer` | 调用大模型生成 `summary` / `highlights` / `tags` / `score`，过滤低质量内容 | `knowledge/raw/` 原始条目 | `knowledge/articles/<date>/*.json`（status=`reviewed`） |
| 整理 | `curator` | 校验字段完整性、合并重复主题、维护索引、触发分发渠道 | `knowledge/articles/`（status=`reviewed`） | 更新后的条目（status=`published`）+ 分发回执 |

各 Agent 的人格定义、提示词、工具白名单见 `.opencode/agents/<name>.md`。

## 7. 红线（绝对禁止）

以下行为视为严重违规，CI/审核 Agent 一旦发现立即阻断、回滚并告警：

1. **不得**将任何 API Key、Token、Webhook 密钥、机器人 Secret 写入仓库（含历史提交）；一律走环境变量或密钥管理。
2. **不得**绕过来源站点的 `robots.txt` 与速率限制；采集频率必须与 `.opencode/skills/` 中声明的节流参数一致。
3. **不得**在未经分析 Agent 评分与去敏的情况下直接对外分发原始抓取内容。
4. **不得**修改 `knowledge/raw/` 下的历史数据（仅追加，不可改写、不可删除）。
5. **不得**对 `main` 分支强推（`git push -f`），不得删除受保护分支。
6. **不得**使用裸 `print()`、`eval`、`exec`、`os.system`、`shell=True` 的 subprocess 调用。
7. **不得**抓取或转发涉及个人隐私、未公开漏洞利用细节、版权受限的全文正文（仅保留链接与摘要）。
8. **不得**让任何 Agent 自行升级自身权限、修改 `AGENTS.md` 或 `.opencode/agents/*` 的红线/权限段落 —— 这类变更必须由人工 PR 审核。
9. **不得**在任何分发渠道发送未标注 `[AI 自动整理]` 来源的内容。
10. **不得**跳过 `ruff` / `mypy` / 单元测试直接合并。

---

> 任何对本文件的修改都必须经过人工 Code Review 并在 PR 描述中说明动机；Agent 自身不得静默改写。
