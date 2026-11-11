> **最新阶段：1H（出站媒体队列与 MinIO 私有存储）**。请查看 [第九阶段1H媒体发送交付指南](docs/第九阶段1H媒体发送交付指南.md)。已完成本地模拟接口测试，真实 Meta、Chatwoot 联调与 Vue 生产构建仍待验证。

> **最新阶段：1F（WebSocket 消息通知）**。新功能和未完成事项见 [第七阶段1F实时消息交付指南](docs/第七阶段1F实时消息交付指南.md)。前端构建及 Meta/Chatwoot 真实联调仍待验证。

# SupplyFlow CRM · Phase 1C

本版本延续原有 1B，新增 Alembic 数据库迁移、分页、管理员 Excel 导入、A 级客户释放审批、跟进提醒、30 天公海自动回收和管理端对应入口。

**使用与升级：** 请先阅读 [第三阶段 1C 交付与升级指南](docs/第三阶段1C交付与升级指南.md)。

# Supply Inquiry CRM / FastAPI + Vue 3 Phase 1B

This is a **backend demonstration and Vue 3 frontend source project**, not a production-ready application. It implements the PRD's phase-one CRM foundation (not the messaging or supplier portals).

## Implemented

**Phase 1B frontend added:** `frontend/` includes login, dashboard, customers, customer detail/edit, grade, followups, public pool, Excel upload/report, and admin account creation. Messaging and suppliers are clearly marked future modules. See [frontend/README.md](frontend/README.md).

- Users: bootstrap admin, admin-created sales/supplier roles, JWT authentication (Argon2 password hashing).
- Customers: create, list, search, update, assign; owner-based API authorization.
- Manual ABCD grading, append-only audit trail.
- VMAP-style followups with communication channel, summary, requirements, stage and next action.
- Public pool: release and claim, A-grade admin-only release, 20-claim cap, 15-day protection, full followup history retained.
- XLSX import with basic schema validation and per-row error reports (max 1000 records / 3 MB).
- FastAPI OpenAPI (`/docs`) and automated API test suite.

## Start locally (SQLite development mode)

Requires Python 3.11+.

```bash
cd supply-crm-fastapi
python -m venv .venv
# Windows: .venv\Scripts\activate
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env # Windows: copy .env.example .env
```

Edit `.env` with a new random `JWT_SECRET` (e.g., `python -c "import secrets;print(secrets.token_hex(32))"`) and a strong 12+ char `ADMIN_PASSWORD`.

```bash
uvicorn app.main:app --reload
```

Visit http://127.0.0.1:8000/docs; log in with the configured admin credentials using the **Authorize** control. Alternatively submit `POST /api/auth/token` with `username` and `password` form fields.

## Docker + PostgreSQL development mode

Set `DATABASE_URL` in `.env` aside (the compose file overrides it), and add `POSTGRES_PASSWORD=<strong password>` to `.env`. Then:

```bash
docker compose up --build
```

API: http://127.0.0.1:8000/docs

## Frontend local startup

After the backend starts, run `cd frontend && npm install && npm run dev`, then open http://localhost:5173. You may also run `docker compose --profile frontend up --build`. Frontend dependencies could not be downloaded in the artifact-building environment, so **no claim of successful frontend compilation is made**.

## Run tests

```bash
pytest -q
```

The test uses a temporary isolated SQLite database and verifies real HTTP workflow and permission behavior.

## API endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/auth/token` | login, form data |
| GET | `/api/auth/me` | current user |
| POST/GET | `/api/users` | admin manages users |
| POST/GET | `/api/customers` | create/list customer |
| GET/PATCH | `/api/customers/{id}` | detail / partial edit |
| POST | `/api/customers/{id}/grade` | manual ABCD with reason |
| POST/GET | `/api/customers/{id}/followups` | followup create / history |
| POST | `/api/customers/{id}/release` | release to public pool |
| POST | `/api/customers/{id}/claim` | salesperson claims public pool customer |
| POST | `/api/customers/{id}/assign` | admin assigns customer |
| GET | `/api/customers/{id}/audit` | immutable action history |
| POST | `/api/import/customers` | sales XLSX import |
| GET | `/api/import/{job_id}` | import failure report |

Example import XLSX header: `company,country,contact_name,phone,whatsapp,email,channel,channel_account,category` (only `company` and `country` mandatory).

## Deliberate scope limits / production checklist

- Vue 3 frontend source code is present for the Phase 1 CRM functions. Chatwoot, WhatsApp and Douyin integrations are **not** implemented. Supplier procurement workflows are **not** implemented. Supplier login is defined, but supplier users have **no CRM/customer access**.
- The PRD's N-day inactivity release is **not** implemented as a scheduled job. Admin and manual release are implemented; A-grade review is modeled as admin-only release rather than a multi-step approval. The claim cap is fixed at 20 and protection is fixed at 15 days in this demo; later move to configuration tables.
- SQLite is only for local tests. Use PostgreSQL for real concurrency. In production, enforce public-pool caps with per-user row locks/advisory locks to avoid concurrent claims bypassing the 20 limit.
- Production requires Alembic migrations, password reset, token revocation/refresh design, rate limiting, audit retention, observability, encrypted backups, test isolation, safe CORS, CSRF approach for cookie auth, robust PII handling, and password policy.
- Customer imports use in-memory parsing and simple validation. Add file scanning, deduplication, full schema validation, chunked/background processing and field-level failure diagnostics before production.
- Admin import is intentionally disabled in this prototype; admins can assign imported sales customers after salesperson import. This restriction is not a PRD requirement and should be implemented in the next iteration.
- Account-based unread notifications, message ingestion, category mapping, push deduplication, quotation visibility and supplier identity binding are future milestones.
- Seed is demo-only: an admin is automatically created on first startup if `ADMIN_PASSWORD` is configured. Rotate and remove startup bootstrap for production.

## Phased architecture

`Vue3 -> FastAPI (auth, customers, pool, follows, demands, suppliers) -> PostgreSQL`;
`WhatsApp official Cloud API / Douyin authorized Connector -> Chatwoot -> unified Vue message UI`;
`Redis + Celery` later for jobs; `MinIO / S3` for supplier files and message media. Chatwoot keeps **its own** PostgreSQL and Redis resources and is integrated using supported APIs/webhooks, not direct shared tables.

## 第四阶段 1D：WhatsApp 官方消息桥接（验证版）

在 1C CRM 后端新增独立 `app/channel_bridge.py`，支持 Meta WhatsApp Cloud API 签名 Webhook、Chatwoot API Channel 账号绑定、文本消息转发、出站人工回复回调、消息去重及汇总指标。

- 数据迁移：`alembic upgrade head`（最新 `0003`）
- 管理端“消息中心”支持查看及绑定账号
- **当前状态**：本地外部 API Mock 测试通过；真实 Meta/Chatwoot 联调未完成，媒体、可靠队列、SSO/红点仍需开发。
- 接入部署、权限及剩余风险：`docs/第四阶段1D消息桥接实施指南.md`

## Phase 1D+（2026-11-10）

新增数据库持久化消息队列、独立投递 worker、Webhook 入队去重、失败重试及死信状态、队列/会话监控接口和前端展示。详见 `docs/第五阶段1D+可靠消息交付指南.md`。升级前请备份数据库并运行 `alembic upgrade head`。该版本**未完成媒体文件传输和真实平台联调**。


## 阶段 1E 消息工作台
新增三栏 WhatsApp 文本工作台、3 秒轮询、账号/会话未读数与人工回复接口。执行 `alembic upgrade head` 升级到 0005。详细见 `docs/第六阶段1E聊天工作台交付指南.md`。

## Phase 1G 增量功能

增加 WhatsApp 入站图片、语音、文档、视频附件的受限下载与向 Chatwoot 上传、受权限控制的消息附件下载、状态回执防降级，数据库迁移至 0006。**不支持出站媒体；未完成真实平台联调。** 详见 `docs/第八阶段1G媒体接收与回执交付指南.md`。

## Windows / Linux 一键部署脚本（新增）

提供 `deploy/windows-docker.bat`、`deploy/windows-native.bat`、`deploy/linux-docker.sh`、`deploy/linux-native.sh`。直接参阅 [`deploy/一键部署说明.md`](deploy/一键部署说明.md)。Docker 方案包括 PostgreSQL + MinIO；不用 Docker 的快捷方案默认使用 SQLite，媒体发送需要外部对象存储。首次运行仍需安装对应的 Docker 或 Python / Node 基础环境。

## 1J：抖音消息中心本地模拟（新增）

消息中心现在提供 WhatsApp 与“抖音 · 本地模拟测试”渠道切换。**抖音功能只供无真实授权时演示**：管理员创建模拟账号、绑定业务员、模拟收到客户文本消息、人工本地回复、未读计数、已读状态及隔离权限。所有模拟操作均不连接抖音，不是官方账号扫码登录。

- 数据库新增 `0007_douyin_demo.py` 迁移，沿用四个原有部署脚本；现有 SQLite 使用修复版 `sqlite_preflight` 避免重复建表。
- 升级流程和业务操作见 [`docs/抖音私信模拟测试与官方接入准备.md`](docs/抖音私信模拟测试与官方接入准备.md)。
- 真实抖音企业私信需要 `enterprise.im` 权限、企业号授权和官方事件/API 联调，目前尚未完成。
