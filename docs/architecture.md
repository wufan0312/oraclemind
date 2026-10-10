# 架构与技术栈

> 本文描述玄镜 OracleMind 的整体架构、技术选型与部署拓扑。源码分布在各子仓（见根 `README.md` 子仓导航）。

## 1. 分层概览

```
用户/浏览器 ──HTTPS──▶ 前端 (Next.js 14)
                        │
            ┌───────────┴────────────┐
            ▼                         ▼
      后端 (FastAPI)             AI 服务 (FastAPI)
      确定性排盘计算              多智能体 + RAG + SSE
            │                         │
       ┌────┴────┐               ┌────┴────┐
       ▼         ▼               ▼         ▼
   Neon PG   Upstash Redis    Upstash   Chroma(可选)
                                  Redis     RAG
```

- **前端**：`oraclemind-frontend`，Next.js 14 App Router + React 18 + TypeScript + Sass 自建组件库（暗色紫青主题，主色 `#7c5ffc`、强调 `#5ce1e6`，报告/成果用金色）。不依赖 AntD/MUI。
- **后端**：`oraclemind-backend`，FastAPI 排盘计算服务，负责确定性计算（八字/紫微/六爻/梅花/奇门/数字命理）；AI 只做解读不做数值计算。SQLAlchemy 2.0 async，PG/SQLite 双数据库策略。
- **AI 服务**：`oraclemind-ai`，LangGraph 多智能体编排 + RAG（chromadb）+ SSE 流式输出，模型智谱 `glm-4-flash`；LLM 失败→重试→本地规则降级（用户无感）。
- **管理后台**：`oraclemind-admin`（`:3002` 前端 + `:8001` 后端），本地运行，尚未纳管远程仓与 Vercel。
- **微信小程序**：`oraclemind-miniprogram`，品牌名「觉境」，尚未纳管。

## 2. 接入与降级原则

前端优先调用后端 API：
- `src/lib/api.ts` 中 `API_BASE` 走环境变量（本地 `localhost:8000` / 生产 `oraclemind-backend.vercel.app`）。
- 后端不可达时前端静默降级本地算法，功能不受阻断。
- `NEXT_PUBLIC_*` 为构建期内联变量，改值必须重新部署。

## 3. 数据与外链依赖

| 依赖 | 用途 | 备注 |
|------|------|------|
| Neon PostgreSQL | 后端主库（生产） | `postgresql+asyncpg://` 驱动，禁用 SQLite |
| Upstash Redis | 缓存 / 限流 | `rediss://`（TLS），不可用时降级内存兜底 |
| Chroma（chromadb） | AI RAG 检索 | 逼近 Vercel 500MB 包体上限，必要时关 `RETRIEVAL_ENABLED` |
| 智谱 AI | `glm-4-flash` 解读 | `AI_API_KEY` 走面板 |

## 4. 部署拓扑（Vercel）

三服务各自作为**独立 Vercel 项目**部署，互不共享配置。部署顺序：**后端 → AI → 前端**（前端构建时需拿到前两着的域名注入 `NEXT_PUBLIC_*`）。

| 项目 | 本地目录 | 形态 | 域名 |
|---|---|---|---|
| oraclemind-frontend | `oraclemind-frontend/` | Next.js 14 | `oraclemind-frontend.vercel.app` |
| oraclemind-backend | `oraclemind-backend/` | Python ASGI | `oraclemind-backend.vercel.app` |
| oraclemind-ai-eight | `oraclemind-ai/` | Python ASGI | `oraclemind-ai-eight.vercel.app` |

`vercel.json` 约定（详细见 `VERCEL_DEPLOY.md`）：
- 顶层只用 schema 允许的键；文件排除一律走 `.vercelignore`。
- 后端 `buildCommand: "alembic upgrade head"`（构建期幂等建表，可删改由冷启动兜底）。
- AI 函数 `maxDuration=60`、`memory=1024`；`CHROMA_PERSIST_DIR` 必须指 `/tmp`。

## 5. 外部依赖链接

- 前端 → 后端：`NEXT_PUBLIC_API_BASE`（不含 `/api`）
- 前端 → AI：`NEXT_PUBLIC_AI_API_BASE`（不含 `/api`）
- 后端 / AI → 前端：`CORS_ORIGINS`（逗号分隔域名）

规则：env 一律填**域名根路径，不含 `/api`**（`/api/v1/...` 由前端请求路径携带）。
