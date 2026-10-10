# 部署与环境变量规范（精炼版）

> 完整排错手册见根目录 `VERCEL_DEPLOY.md`；变量分层规范见 `ENV_SETUP.md`。本文为速查版。

## 1. 前置依赖（一次性）

- **Vercel CLI**：`npm i -g vercel` → `vercel login`（无法弹浏览器时用 token：`vercel.com → Settings → Tokens`）。
- **Neon**（PostgreSQL）：region 选 Singapore；连接串改前缀为 `postgresql+asyncpg://`。
- **Upstash**（Redis）：`rediss://`（注意是 `rediss`，Vercel 走 TLS）。

## 2. 部署顺序：后端 → AI → 前端

```bash
# 后端
cd oraclemind-backend && vercel link && vercel deploy --prod
# AI
cd oraclemind-ai && vercel link && vercel deploy --prod
# 前端（必须先配 NEXT_PUBLIC_* 再 deploy，且改值必须重新部署）
cd oraclemind-frontend && vercel link && vercel deploy --prod
```

## 3. 必配环境变量

**后端（oraclemind-backend）**

| 变量 | 必填 | 说明 |
|---|---|---|
| `DATABASE_URL` | ✅ | `postgresql+asyncpg://`，SSL 参数用 `ssl=require`（非 `sslmode`），pooler 主机加 `prepared_statement_cache_size=0` |
| `JWT_SECRET` | ✅ 生产 | 空值/默认 `change-me` 一律拒绝启动 |
| `ENCRYPTION_KEY` | ✅ | Fernet key；换值导致旧密文不可解 |
| `CORS_ORIGINS` | ✅ | 逗号分隔前端域名 |
| `APP_ENV` | 建议 | `production` |
| `REDIS_URL` | 可选 | 不配降级内存 |

**AI（oraclemind-ai）**：`AI_API_KEY`（智谱）、`AI_MODEL_CHAT`/`AI_MODEL_LITE`=`glm-4-flash`、`AI_BASE_URL`、`APP_ENV=production`、`RETRIEVAL_ENABLED`（生产建议 `false`）、`CHROMA_PERSIST_DIR=/tmp/...`、`REDIS_URL`、`CORS_ORIGINS`。

**前端（oraclemind-frontend）**：`NEXT_PUBLIC_API_BASE`、`NEXT_PUBLIC_AI_API_BASE`（均不含 `/api`）、`NEXT_PUBLIC_ICONFONT_CDN`。

## 4. 三个「静默坑」（必读）

1. **面板空串变量盖文件**：面板变量建了却留空，比不建更危险（构建内联 `""`）。用只在文件里定义、面板绝不配的变量（如 `NEXT_PUBLIC_ICONFONT_CDN`）去产物查是否生效来判别。
2. **`.vercelignore` 误排分层配置**：不要写 `.env.*` 通配（会连 `.env.production` 排除）。精确列举 `.env` / `.env.local` / `.env.*.local`。
3. **改面板变量不触发部署**：必须手动 Redeploy（或再 push 一个 commit）。

## 5. 一键验收

```bash
curl -s https://oraclemind-backend.vercel.app/api/health      # database: up
curl -s https://oraclemind-ai-eight.vercel.app/health        # llm_available: true
curl -sI -H 'Origin: https://oraclemind-frontend.vercel.app' \
     https://oraclemind-backend.vercel.app/api/health | grep -i access-control-allow-origin
```

## 6. 判错口诀

- `{"detail":"Not Found"}` / `{"error":"SERVER_ERROR",...}` → **应用活着**，只是路径/业务问题。
- `FUNCTION_INVOCATION_FAILED` / "This Serverless Function has crashed" → **函数进程没起来**（依赖缺失/import 期 raise，见 `VERCEL_DEPLOY.md` §6.5）。
- `DEPLOYMENT_NOT_FOUND` → 域名下无部署（项目名/域名不对）。
- 只要响应体是 JSON 且带 `detail`/`error`，就说明应用在跑。

## 7. 推送提醒

- 根主仓库的 push **不触发任何部署**（Vercel 连的是各子仓）。改哪个服务就在哪个子仓提交。
- remote 统一 SSH；`github.com` 受限时走代理或 Vercel CLI 直连部署。
