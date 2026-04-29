# 项目结构说明（开发者导航）

本文档面向**二次开发与维护**：说明仓库目录边界、调用链入口、以及常见改动应落在何处。更偏用户向的说明见根目录 [README.md](../README.md) 与 [docs/full-guide.md](full-guide.md)。

---

## 1. 项目定位与调用方

**Daily Stock Analysis** 是一套 A股/港股/美股的自选股智能分析系统：拉取行情与新闻 → 技术面/策略辅助 → 通过 LiteLLM 调用大模型生成报告 → 多渠道推送，并支持 Web API、Bot、定时任务（GitHub Actions）。

典型**调用方**与关系：

| 调用方 | 入口/文件 | 与核心逻辑的关系 |
|--------|-----------|------------------|
| CLI 定时/本地跑批 | `main.py` | 解析参数 → `StockAnalysisPipeline` / `run_market_review` |
| 可被其他模块复用的分析封装 | `analyzer_service.py` | 薄封装，内部仍用 `StockAnalysisPipeline` |
| HTTP API | `server.py` → `api/app.py` → `api/v1/` | FastAPI 路由调用 `src/services/*` 与流水线 |
| 仅启动 Web 后端 | `webui.py` 或 `python main.py --webui-only` | 同 `api.app:app` |
| 钉钉/飞书等 Bot | `bot/handler.py`、`bot/dispatcher.py`、`bot/commands/*` | Webhook → 命令 → 流水线 / Agent |
| Agent 问股 | `src/agent/*`、`api/v1/endpoints/agent.py` | `build_agent_executor()` 统一构建执行器 |

**核心编排类**：`src/core/pipeline.py` 中的 `StockAnalysisPipeline`（数据、搜索、分析、通知的主流程）。

---

## 2. 顶层目录一览

```
daily_stock_analysis/
├── main.py                 # CLI 主入口（分析、大盘复盘、可选带 API）
├── server.py               # API 服务入口（uvicorn 用 `server:app`）
├── webui.py                # 便捷启动 Web/API
├── analyzer_service.py     # 供 CLI/Web/Bot 复用的分析函数封装
├── api/                    # FastAPI 应用与 v1 REST
├── bot/                    # 多平台 Bot：命令、分发、平台适配
├── src/                    # 业务核心：配置、流水线、Agent、存储、通知等
├── data_provider/          # 行情数据源策略与 DataFetcherManager
├── strategies/             # Agent 用 YAML 自然语言策略（非 Python 代码）
├── tests/                  # pytest
├── docs/                   # 文档、部署说明
├── docker/                 # Compose 等
├── scripts/                # 构建脚本（如桌面/macOS）
├── apps/                   # 前端 SPA（dsa-web）与桌面壳（dsa-desktop）
├── patch/                  # 第三方补丁（如 eastmoney）
├── templates/              # 报告等模板
├── static/                 # 构建后的前端静态资源（由构建流程产出）
├── requirements.txt        # Python 依赖
├── pyproject.toml          # black / isort / bandit 等工具配置
├── test.sh                 # 本地场景化测试脚本
├── .env.example            # 环境变量示例（实际配置用 .env）
└── AGENTS.md               # 本仓库协作与 Issue/PR 规范
```

---

## 3. `src/` — 核心业务层（改功能多半在这里）

| 路径 | 职责 |
|------|------|
| `src/config.py` | 从 `.env` 加载配置；`Config` dataclass；`setup_env()` / `get_config()` |
| `src/core/pipeline.py` | **分析主流程**：并发、单股处理、与 fetcher/search/analyzer/notifier 协作 |
| `src/core/market_review.py` | 大盘复盘流程 |
| `src/core/market_strategy.py`、`market_profile.py` | 市场状态/策略框架（A股三段式、美股 Regime 等） |
| `src/core/backtest_engine.py` | 回测引擎逻辑 |
| `src/core/trading_calendar.py` | 交易日、市场归属等 |
| `src/core/config_manager.py`、`config_registry.py` | 运行时策略/配置注册（与 YAML 策略配合） |
| `src/analyzer.py` | LLM 分析封装（与 LiteLLM、提示词、结果结构相关） |
| `src/stock_analyzer.py`、`market_analyzer.py` | 技术面/市场文本生成等 |
| `src/search_service.py` | 新闻/搜索聚合（Tavily、SerpAPI 等） |
| `src/notification.py` | 通知调度，组合各 `notification_sender` |
| `src/notification_sender/` | 各渠道具体发送实现（微信、飞书、Telegram、邮件等） |
| `src/storage.py` | SQLite + SQLAlchemy ORM、报告与历史等持久化 |
| `src/repositories/` | 对存储的仓储封装（analysis、backtest、stock 等） |
| `src/services/` | 面向 API/Bot 的服务层：分析任务、历史、回测、导入解析、报表渲染、系统配置等 |
| `src/agent/` | Agent：`executor`、`conversation`、`llm_adapter`、`factory` |
| `src/agent/tools/` | Agent 可调工具（行情、分析、搜索、市场类），在 `registry` 注册 |
| `src/agent/skills/` | Skill 基类；具体策略内容多在 `strategies/*.yaml` |
| `src/schemas/` | 报告等 JSON Schema |
| `src/data/` | 静态映射等（如 `stock_mapping`） |
| `src/utils/` | 通用数据处理等 |
| `src/auth.py` | 认证相关（API/Bot 共用部分逻辑） |
| `src/webui_frontend.py` | 前端资源准备（与静态站点配合） |
| 其他 | `feishu_doc.py`、`md2img.py`、`formatters.py`、`scheduler.py`、`logging_config.py`、`enums.py` 等 |

**修改建议**：

- 改「跑一次分析的步骤、并发、失败重试」→ `pipeline.py`
- 改「提示词/模型调用/解析 AI 返回」→ `analyzer.py`、`llm_adapter.py`
- 改「数据源优先级或单源实现」→ `data_provider/`
- 改「REST 行为或请求校验」→ `api/v1/endpoints/` 与 `api/v1/schemas/`
- 改「Bot 指令与回复」→ `bot/commands/`

---

## 4. `data_provider/` — 行情数据

- `base.py`：`BaseFetcher`、`DataFetcherManager`（故障切换、优先级）
- 各 `*_fetcher.py`：Efinance、AkShare、Tushare、Pytdx、Baostock、YFinance 等
- `realtime_types.py`：实时/筹码等类型定义
- `us_index_mapping.py`：美股指数代码映射

包级 `__init__.py` 中有**数据源优先级**说明，改默认顺序或新增源时应对齐该文档注释。

---

## 5. `api/` — FastAPI

| 路径 | 职责 |
|------|------|
| `api/app.py` | `create_app()`：CORS、静态文件、生命周期、`SystemConfigService` |
| `api/v1/router.py` | 聚合 v1 子路由，前缀 `/api/v1` |
| `api/v1/endpoints/` | `analysis`、`auth`、`history`、`stocks`、`backtest`、`system_config`、`agent`、`usage`、`health` |
| `api/v1/schemas/` | Pydantic 请求/响应模型 |
| `api/deps.py` | 依赖注入（如 DB、配置） |
| `api/middlewares/` | 认证、统一错误处理 |

生产静态前端默认目录：`api/app.py` 中相对于项目根的 `static/`（与 `apps/dsa-web` 构建产物衔接）。

---

## 6. `bot/` — 聊天机器人

| 路径 | 职责 |
|------|------|
| `bot/handler.py` | 统一 Webhook 入口，`handle_webhook()` |
| `bot/dispatcher.py` | 命令解析、频率限制、`CommandDispatcher` |
| `bot/commands/` | 各子命令：`analyze`、`chat`、`ask`、`market`、`batch` 等 |
| `bot/platforms/` | 钉钉、飞书流式等适配；`base` 定义接口 |
| `bot/models.py` | `BotMessage`、`BotResponse` 等 |

新增命令：实现 `BotCommand` 子类并注册到 dispatcher（参见现有 `commands`）。

---

## 7. `strategies/` — Agent 策略 YAML

- 每个 `.yaml` 描述一条可激活的「技能/策略」（自然语言 + 可选工具列表等）
- 加载与 Skill 管理逻辑在 `src/agent/` 与 `src/core/config_registry.py` 等相关模块
- 详见 [strategies/README.md](../strategies/README.md)

---

## 8. `apps/` — 前端与桌面

| 目录 | 说明 |
|------|------|
| `apps/dsa-web/` | 现代 Web 前端（Vite 等），构建产物通常同步到根目录 `static/` |
| `apps/dsa-desktop/` | 桌面应用壳（如 Electron/Tauri 类结构），与 `scripts/build-desktop-macos.sh` 等配合 |

**纯后端改动**通常不必动 `apps/`；若改 API 契约，需同步前端调用。

---

## 9. 配置与运行

- **环境变量**：统一 `.env`，模板见 `.env.example`；支持 `ENV_FILE` 指定路径
- **代码中读取**：`src/config.py` 的 `Config` / `get_config()`
- **常用启动**：
  - `python main.py` — 完整 CLI 流程
  - `uvicorn server:app --host 0.0.0.0 --port 8000` — API
  - `python webui.py` — 等价于拉起 `api.app:app`

---

## 10. 测试与质量

- **单元/集成测试**：`tests/`，pytest
- **快捷脚本**：`./test.sh`（多种场景：`quick`、`dry-run`、`us-stock` 等）
- **规范**（见 `AGENTS.md`）：行宽 120、`black` + `isort` + `flake8`；可运行 `./test.sh syntax` 或 `python -m py_compile ...`

---

## 11. CI / Docker / 文档

- `.github/workflows/`：`ci.yml`、`daily_analysis.yml`、`docker-publish.yml` 等
- `docker/docker-compose.yml`：容器编排
- `docs/`：部署（`DEPLOY.md`）、Docker（`docs/docker/`）、Bot 配置（`docs/bot/`）、变更日志（`CHANGELOG.md`）

---

## 12. 按需求快速定位（速查）

| 你想做的事 | 优先看的文件/目录 |
|------------|-------------------|
| 调整分析流程顺序或并发 | `src/core/pipeline.py` |
| 换模型/渠道/降级 | `src/config.py`、`src/analyzer.py`、`src/agent/llm_adapter.py` |
| 新增或修改数据源 | `data_provider/*_fetcher.py`、`data_provider/base.py` |
| 新增 HTTP 接口 | `api/v1/endpoints/`、`api/v1/schemas/`、`api/v1/router.py` |
| Bot 新指令 | `bot/commands/`、`bot/dispatcher.py` |
| Agent 新工具 | `src/agent/tools/`、`src/agent/tools/registry.py`、`src/agent/factory.py` |
| 新增推送渠道 | `src/notification_sender/`、`src/notification.py` |
| 数据库字段/历史报告结构 | `src/storage.py`、`src/repositories/` |
| 回测逻辑 | `src/core/backtest_engine.py`、`src/services/backtest_service.py` |
| 仅文档/发布流程 | `docs/`、`AGENTS.md`、`.github/workflows/` |

---

*文档生成自仓库当前结构梳理；若目录有增删，请以代码为准并酌情更新本节。*
