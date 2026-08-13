# AGENTS.md — 给 AI 看的项目交接文档

> 本文件是本仓库最重要的文档。任何新对话/AI 在本仓库工作前，必须完整阅读本文件，并阅读 `docs/` 中与本任务相关的文档。
> 建议阅读顺序：本文档 → `docs/PRODUCT_SPEC.md` → `docs/ARCHITECTURE.md` → 任务相关的专项文档（DATA_MODEL / IMPORT_AND_CLASSIFICATION / ROADMAP）。

## 1. 项目定位（一句话）

一个自托管的个人记账 Web 应用：支持微信/支付宝账单导入、自动分类、同笔资金识别（退款/当天还/AA 分账），并生成周/月/自定义时间段报表；先做网页，后续扩展到手机端。

## 2. 当前状态（重要）

- 阶段：**阶段 0（文档初始化）**。仓库里还没有任何应用代码。
- 已有内容：`README.md`、`AGENTS.md`、`docs/` 全套设计文档、`data/`（预留目录）。
- 不要假设已有功能；任何"实现某功能"的任务都从阶段 1 开始规划，并先阅读 [docs/ROADMAP.md](docs/ROADMAP.md)。

## 3. 技术栈与目录约定（已决策，不要改）

| 层 | 技术 | 说明 |
|----|------|------|
| 前端 | Vue 3 + TypeScript + Vite + Pinia + Element Plus | 先做响应式网页，后续 PWA |
| 后端 | Python 3.12 + FastAPI + SQLAlchemy 2 + Alembic + Pydantic v2 | REST API，前缀 `/api/v1` |
| 数据库 | PostgreSQL 16 | 主存储；迁移必须用 Alembic |
| 部署 | Docker Compose（frontend/nginx + backend + postgres） | 自托管；HTTPS 建议 Caddy |
| 测试 | pytest（后端）、Vitest（前端），后续 E2E | 每阶段验收前置 |
| 代码质量 | Ruff（后端）、ESLint + Prettier（前端） | CI 中强制执行 |

目录规划（阶段 1 起逐步创建）：

- `frontend/`：Vue 3 前端（阶段 1 用官方脚手架生成）
- `backend/`：FastAPI 后端（阶段 1 起）
- `docs/`：全部设计文档（本阶段已建好）
- `data/`：预留数据目录（暂不存业务数据；生产数据在 PostgreSQL）

## 4. 硬性纪律（违反即视为"犯病"）

1. **分层**：前端组件禁止写业务逻辑；后端必须 controller(路由) → service(业务) → repository(数据访问) 分层。禁止在路由函数里堆业务代码。
2. **金额**：所有金额一律以整数"分"存储和计算（`amount_cents`），禁止 float。展示时再换算成元。
3. **时间**：数据库存 UTC（`timestamptz`）；API 传 ISO 8601 带时区；界面按用户本地时区显示。禁止存无时区的本地时间。
4. **数据库变更**：必须同时做三件事——写 Alembic 迁移、更新 `docs/DATA_MODEL.md`、更新受影响的测试。
5. **分类可追溯**：任何自动分类结果必须记录来源（rule / merchant_memory / ai / manual）与置信度；禁止静默覆盖用户确认过的分类；用户手动修正优先级最高。
6. **密钥安全**：密钥/令牌只走环境变量（`.env`，已被 .gitignore 忽略），禁止硬编码、禁止提交到仓库。
7. **依赖纪律**：只引入知名、活跃维护、有安全响应的库；锁定版本；禁止"三无"小工具库。
8. **先文档后代码**：新增/修改功能前，先更新对应的 `docs/` 文档与 ROADMAP 阶段，再写代码。
9. **测试**：任何功能必须有对应测试；回归到 CI。
10. **隐私红线**：账单数据不出自有服务器；AI 调用只发送最少必要字段（商户名、金额、时间、交易类型），且可配置关闭；AI 请求日志必须脱敏。
11. **不越权**：不要实现计划外的大功能；遇到不确定的产品决策，先查 `docs/PRODUCT_SPEC.md` 的既定规则；若仍无答案，明确询问用户而不是擅自假设。

## 5. 统一术语表

| 术语 | 英文/标识 | 含义 |
|------|-----------|------|
| 账本/账户 | Account | 资金来源，如微信、支付宝、现金（每个支付平台一个账户） |
| 交易/账单 | Transaction | 一笔收支记录 |
| 类目 | Category | 分类，如餐食、交通；用户可自定义 |
| 商户 | Merchant | 交易对方（如"李掌柜烧饼店"）；存储归一化名称 |
| 记事谱 | MerchantRule | 商户名 → 类目的记忆表；用户确认过的条目优先级最高 |
| 对账组 | MatchGroup | 同笔资金的关联组（退款/当天还/AA 分账），确认后生效 |
| 导入批次 | ImportBatch | 一次导入的账单文件及解析统计 |
| 报表 | Report | 周/月/自定义时间段的统计视图 |

## 6. 常见坑（最容易犯的错）

- 微信账单和支付宝账单字段名不同：必须走"字段映射表"（见 `docs/IMPORT_AND_CLASSIFICATION.md`），不要按一个平台的表头硬写。
- 退款不是支出；AA 收款不是收入：识别确认后按"净支出"统计，不能重复计入。
- 微信/支付宝导出文件常见 GBK 编码，必须转 UTF-8；金额单位是"元"（可能带负号），入库前转"分"。
- 同一笔交易可能重复导入：必须用去重键（平台+交易单号+商户单号+金额+时间）。
- "记事谱"命中时直接归类、不调 AI；AI 只在规则/记事谱都拿不准时兜底（见分类流水线）。
- 不要用"当前时间"代替交易发生时间：每笔交易有自己的 `occurred_at`。

## 7. 新增功能的固定工作流

1. 读 AGENTS.md + `docs/PRODUCT_SPEC.md` + `docs/ARCHITECTURE.md` + 相关专项文档；
2. 确认该功能属于哪个 ROADMAP 阶段；若跨越多个阶段，先与用户确认顺序；
3. 更新 docs（产品规格/数据模型/架构）后再动代码；
4. 实现 + 测试 + 更新 ROADMAP 完成状态；
5. 提交信息遵循 Conventional Commits（feat/fix/docs/refactor/test）。

## 8. 运行方式（规划，待阶段 1 落地）

- 开发：后端 uvicorn 热重载 + 前端 vite dev（代理 /api 到后端）。
- 生产：`docker compose up -d`（frontend/nginx 反代 /api 与静态资源；postgres 持久化）。
- 初始化：Alembic 迁移建表；首次启动创建默认类目（餐食、交通、购物、居住、娱乐、医疗、转账往来、其他等）。
