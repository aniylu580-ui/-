# 技术架构（Architecture）

## 1. 总体架构

```text
手机浏览器 / PWA（后续原生 App）
        │ HTTPS
        ▼
┌─────────────────┐   REST /api/v1   ┌──────────────────────┐
│ 前端 Vue 3       │ ──────────────▶ │ 后端 FastAPI          │
│ Nginx 静态托管    │ ◀────────────── │ 路由 → 服务 → 仓储     │
└─────────────────┘   JSON           └──────────┬───────────┘
                                                │ SQLAlchemy
                                                ▼
                                        ┌──────────────┐
                                        │ PostgreSQL 16 │
                                        └──────────────┘
                              外部：可插拔 AI 网关（可配置关闭）
```

## 2. 技术选型与理由

| 组件 | 选择 | 理由 |
|------|------|------|
| 前端框架 | Vue 3 + TypeScript + Vite | 社区最活跃的主流方案；TS 保证可维护性 |
| UI 组件 | Element Plus | 成熟的中后台组件库，中文文档完善 |
| 状态管理 | Pinia | Vue 官方推荐，类型友好 |
| 后端框架 | FastAPI + Pydantic v2 | 类型安全、自动 OpenAPI 文档、性能好 |
| ORM/迁移 | SQLAlchemy 2 + Alembic | 业界标准；迁移可审计 |
| 数据库 | PostgreSQL 16 | 开源、功能强、适合财务数据 |
| 部署 | Docker Compose | 自托管简单、环境一致 |
| 反向代理 | Nginx（前端） + Caddy（HTTPS，可选） | 成熟安全 |
| AI 接入 | 可插拔 AI 网关（OpenAI 兼容接口） | 可接 DeepSeek/通义千问/本地模型，密钥走环境变量，可配置关闭 |

选型原则：只用知名、活跃维护、有安全响应的工具；不引入"三无"库。

## 3. 分层与模块

后端按业务域拆分模块（`backend/app/` 下）：

- `auth`：登录/鉴权（v1 单用户，先做简单令牌；多用户时再补）
- `accounts`：账户/账本
- `categories`：类目
- `merchants`：商户与记事谱（MerchantRule）
- `transactions`：交易（增删改查、搜索）
- `imports`：账单导入（解析、去重、批次）
- `matching`：同笔资金识别（退款/当天还/AA）
- `reports`：报表聚合
- `ai`：AI 网关（统一封装，可插拔、可关闭）
- `audit`：审计日志

分层规则：

- 路由层（controller）：参数校验、调用 service、返回 DTO；禁止业务逻辑
- 服务层（service）：业务规则（分类流水线、匹配规则、报表口径）
- 仓储层（repository）：SQLAlchemy 数据访问；禁止业务规则
- 前端组件禁止写业务逻辑；业务状态用 Pinia store

## 4. API 规范

- 前缀 `/api/v1`；REST 风格；JSON；统一错误格式 `{ "error": { "code", "message", "detail" } }`
- 分页：列表接口返回 `{ items, total, page, page_size }`
- 时间：ISO 8601 带时区（RFC 3339）
- 金额：以"分"为整数传输（字段 `amount_cents`）；展示由前端换算
- 鉴权：Bearer Token（v1 单用户）；安全测试阶段补 OAuth/CSRF 细节

## 5. 数据访问与迁移

- 所有表结构变更：新建 Alembic 迁移 + 更新 `docs/DATA_MODEL.md`
- 迁移在 CI 中验证（迁移到最新 + 回滚测试）

## 6. 部署（Docker Compose，规划）

```text
services:
  frontend: Nginx 托管构建产物，反代 /api → backend
  backend:  uvicorn 多进程；健康检查 /healthz
  postgres: 数据卷持久化；备份脚本（pg_dump + 加密）
```

- 生产强制 HTTPS（Caddy 自动证书或已有反代）
- 备份：每日 pg_dump，备份文件加密存储，保留 30 天

## 7. 安全基线（先设计，安全测试阶段补全/验证）

- 密码/令牌：argon2 哈希；JWT 短期有效期；登录限流
- API：统一鉴权、CSRF 防护（Cookie 场景）、输入校验（Pydantic）、文件上传类型/大小限制
- 依赖：锁定版本 + 定期安全扫描（pip-audit / npm audit）
- 日志：结构化；AI 请求脱敏；审计日志记录关键操作（导入、删除、分类修改、匹配确认）
- 隐私：账单数据不出自有服务器；AI 功能默认可关闭

## 8. 手机端扩展策略

- 阶段 1 前端即做响应式布局；PWA 允许"添加到主屏幕"
- 后续原生 App（Flutter/uni-app）直接复用 `/api/v1`，不改后端
- 付费/充值（微信/支付宝官方 SDK）属于后续阶段，接口按标准订单流程预留

## 9. 测试策略

- 后端：pytest（服务层单测 + API 集成测试，用测试数据库）
- 前端：Vitest（store/工具函数）+ 组件测试
- 核心算法（解析、去重、分类流水线、匹配规则）必须有单元测试覆盖边界
- 每阶段验收前置：CI 绿 + 文档同步
