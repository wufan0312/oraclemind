# 玄镜 · 环境变量配置规范

三个子项目（前端 / 后端 / AI 服务）统一采用**「按环境自动切换」**的配置方案：
本地开发自动用 `localhost`，生产自动用线上域名，**上线无需再手工改任何地址**。

---

## 一、统一的优先级规则

三端遵循同一条优先级链（高 → 低）：

```
1. 平台注入的真实环境变量   （Vercel 项目面板 / 终端 export）
2. .env.<环境>              （随仓库提交，非敏感默认值，团队共享）
3. .env                     （个人本地兜底，gitignore，放密钥与本机路径）
```

**推论（记住这两条就够）**：

| 场景 | 你要做的事 |
| --- | --- |
| 改「服务地址 / CDN / 端口」这类非敏感配置 | 直接改 `.env.development` / `.env.production`，提交即可 |
| 配「API Key / 数据库密码 / JWT 密钥」 | 本地写 `.env`，线上写 Vercel 面板 —— **不要写进任何已提交的文件** |

> ⚠️ **这条链有个反直觉的后果，本次上线就是栽在这里**：
> `.env.production` 里写对了**没有用** —— 只要 Vercel 面板存在同名变量（哪怕是个错值），**面板值永远赢**。
> 实例：文件里 `CORS_ORIGINS` 是正确的 `oraclemind-frontend.vercel.app`，面板里配的却是
> `oraclemind.vercel.app`（同名撞车的第三方站点），结果线上 CORS 全线拒绝，而本地怎么测都是好的。
> **排查线上配置问题时，第一个该看的是面板，不是仓库。**

---

## 二、文件清单

| 项目 | 切换依据 | 已提交（共享） | 不提交（个人 / 密钥） |
| --- | --- | --- | --- |
| `oraclemind`（前端） | `NODE_ENV`（`npm run dev` / `npm run build` 自动决定） | `.env.development`、`.env.production`、`.env.example` | `.env`、`.env.local`、`.env.*.local` |
| `oraclemind-backend` | `APP_ENV`（默认 `development`） | `.env.development`、`.env.production`、`.env.example` | `.env`、`.env.local`、`.env.prod` |
| `oraclemind-ai-py` | `APP_ENV`（默认 `development`） | `.env.development`、`.env.production`、`.env.example` | `.env`、`.env.local`、`.env.prod` |

> 前端特殊：Next.js 不用 `APP_ENV`，而是内建的 `NODE_ENV` —— `dev` 命令是 development，
> `build`/`start` 与 Vercel 构建是 production。所以前端**不需要**手工设任何开关。

---

## 三、各端要点

### 前端 `oraclemind`

```bash
npm run dev:daemon   # → NODE_ENV=development → .env.development（localhost:8000 / :8021）
npm run build        # → NODE_ENV=production  → .env.production（线上域名）
```

⚠️ **两个必知的坑**：

1. `NEXT_PUBLIC_*` 是**构建期内联**变量。改完必须重新构建；Vercel 上改面板变量后必须 **Redeploy**，
   否则线上产物里仍是旧值（这次 AI 服务不可用的直接原因就是这个）。
2. `.env.local` 在**任何环境下都会被加载**（production 构建也不例外）。
   因此它只能放「仅本人生效」的覆盖项 —— **绝不能写 localhost**，否则本地 `npm run build`
   产出的包会连到你本机的服务。开发默认值统一放 `.env.development`。

### 后端 `oraclemind-backend`

`app/core/config.py` 的 `SettingsConfigDict` 使用 `env_file=(.env, .env.<APP_ENV>)`，
pydantic-settings 保证「环境变量 > env 文件」且「元组靠后的文件覆盖靠前的」。

```bash
APP_ENV=production uvicorn app.main:app --host 0.0.0.0 --port 8000
```

- 生产**禁用 SQLite**：`init_db()` 里 `validate_for_runtime()` 会在 `is_production and is_sqlite` 时直接 raise。
  所以 Vercel 面板**必须**配 `DATABASE_URL`（`postgresql+asyncpg://...`），否则服务起不来 —— 这是刻意的保护。
- 必配：`DATABASE_URL`、`JWT_SECRET`、`ENCRYPTION_KEY`；选配：`REDIS_URL`（不配降级为内存缓存）。

### AI 服务 `oraclemind-ai-py`

`src/config.py` 用 `dotenv_values` 先合并文件、再只写入 `os.environ` 中**尚不存在**的键，
从而实现「面板变量永远最高优先级」。

```bash
APP_ENV=production python scripts/ai_server.py start
```

- 生产**必须**在面板配 `AI_API_KEY`，否则 `/health` 的 `llm_available` 为 false，
  解读会返回降级文案「AI 解读暂时不可用」。
- `AI_SERVICE_API_KEY` **留空**（设了就要求前端带 `X-API-Key` 头）。
- 生产 `.env.production` 把 `RETRIEVAL_ENABLED` 设为 `false`：Vercel 是只读 + 临时文件系统，
  Chroma 索引每次冷启动都要重建，既拖慢冷启动又逼近 500MB 部署体积上限。
- `AI_TOTAL_TIMEOUT_MS`：本地 90000，生产 **55000** —— 因为 `vercel.json` 里
  `api/index.py` 的 `maxDuration` 是 60s，超时会被平台掐断。两边刻意不同，别统一。

---

## 四、上线检查清单

**AI 服务（`oraclemind-ai-eight`）**

- [ ] 面板配 `AI_API_KEY`（智谱 Key）
- [ ] 面板配 `APP_ENV=production`
- [ ] 面板配 `CORS_ORIGINS=https://oraclemind-frontend.vercel.app`
- [ ] 确认 `/health` 返回 `llm_available: true`

**后端（`oraclemind-backend`，2026-09-15 已上线）**

生产域名：`https://oraclemind-backend.vercel.app`

- [x] 建 Vercel 项目，面板配 `APP_ENV=production`
- [x] 面板配 `DATABASE_URL`（`postgresql+asyncpg://`）—— 已生效，`/api/health` 报 `database: up`
- [ ] 面板配 `JWT_SECRET`、`ENCRYPTION_KEY`（非默认占位符，否则生产起不来）
- [ ] 面板配 `CORS_ORIGINS=https://oraclemind-frontend.vercel.app`
- [x] 域名已回填前端 `.env.production` 的 `NEXT_PUBLIC_API_BASE`
      与 AI 服务 `.env.production` 的 `PAIPAN_API_BASE`
- [ ] 补 `REDIS_URL`（Upstash `rediss://`，带 TLS）—— 当前 `/api/health` 报 `redis: down`，
      已降级内存缓存不影响功能，但多实例间缓存不共享、冷启动必穿透
- [ ] 补 `PAYMENT_CALLBACK_SECRET`（否则 `/premium/notify`、`/donations/notify` 一律 403）

**前端（`oraclemind-frontend`）**

- [ ] 确认 `.env.production` 里两个地址正确
- [ ] **Redeploy**（不是改完变量就完事，`NEXT_PUBLIC_*` 必须重新构建）

> 部署顺序：先 AI / 后端，确认 `/health` 正常后，最后再 Redeploy 前端。

---

## 五、本地自检

改完配置想确认加载是否符合预期，可以这样做：

```bash
# 后端
cd oraclemind-backend
python -c "from app.core.config import Settings; s=Settings(); print(s.app_env, s.cors_origin_list)"
APP_ENV=production python -c "from app.core.config import Settings; s=Settings(); print(s.app_env, s.cors_origin_list)"

# AI 服务
cd oraclemind-ai-py
python -c "from src.config import config; print(config.app_env, config.cors_origins, config.total_timeout_ms)"
APP_ENV=production python -c "from src.config import config; print(config.app_env, config.cors_origins, config.total_timeout_ms)"
```

前端没有依赖可跑时，可直接在构建产物里确认内联值：

```bash
npm run build && grep -ro "NEXT_PUBLIC_API_BASE[^\"]*" .next/static/chunks | head -3
```

---

## 六、生产域名对照表（2026-09-15 实测）

| 服务 | 生产域名 | 状态 |
| --- | --- | --- |
| 前端 | `https://oraclemind-frontend.vercel.app` | ✅ 200，标题「玄镜 OracleMind」 |
| 后端 | `https://oraclemind-backend.vercel.app` | ✅ 200，`/api/health` → `database: up` |
| AI 服务 | `https://oraclemind-ai-eight.vercel.app` | ✅ 200，但 `llm_available: false`（缺 `AI_API_KEY`） |

**🚫 域名撞车警告**：`https://oraclemind.vercel.app` **不是我们的项目**，
它是一个英文站（`<title>OracleMind — AI-Powered Life Forecasts</title>`），与我们同名。
配置 `CORS_ORIGINS` / `NEXT_PUBLIC_*` 时**极易误填成它**，且填错不会报错、只会静默拒绝跨域。
填之前先 `curl -s <域名> | grep "<title>"` 确认归属。

**别用 preview 域名做服务地址**：形如 `xxx-hash-user.vercel.app`（如
`oraclemind-backend-hpstdbt0v-wu-fan.vercel.app`）的是预览部署，默认开着 **Vercel SSO**，
任何请求（含服务端 fetch）都会被 302 到 `vercel.com/sso-api`。
判据：`curl -sI <域名>/api/health` 看到 `302` + `Location: https://vercel.com/sso-api`。
要关掉需到项目 Settings → Deployment Protection → 关闭；**正式联调用生产域名更省事**。

### 上线后一键验收

```bash
# 1. 后端存活 + 数据库
curl -s https://oraclemind-backend.vercel.app/api/health
#    期望 {"status":"ok","database":"up","redis":"up"}（redis down 只降级，不致命）

# 2. 后端 CORS 放行前端（有 ACAO 头 = 放行，无 = 拒绝）
curl -sI -H 'Origin: https://oraclemind-frontend.vercel.app' \
     https://oraclemind-backend.vercel.app/api/health | grep -i access-control-allow-origin

# 3. AI 服务 LLM 就绪
curl -s https://oraclemind-ai-eight.vercel.app/health
#    期望 llm_available: true

# 4. AI 服务 CORS 放行前端
curl -sI -H 'Origin: https://oraclemind-frontend.vercel.app' \
     https://oraclemind-ai-eight.vercel.app/health | grep -i access-control-allow-origin

# 5. 前端产物里的地址是否正确（必须 Redeploy 后查）
curl -s https://oraclemind-frontend.vercel.app/ -o /tmp/f.html
grep -o '/_next/static/chunks/[^"]*\.js' /tmp/f.html | head -10
#    下载这些 chunk 后 grep oraclemind-backend / oraclemind-ai-eight 确认已内联
```

---

## 七、部署链路的三个「静默坑」

这三个坑的共同特征：**本地怎么测都是对的，线上静默用默认值，且不报错**。

### 坑 1：Vercel 面板的「空串变量」会盖掉 `.env.production`

面板里建了 `NEXT_PUBLIC_API_BASE` 却**留空**，比不建更危险。

- 原理：`@next/env` 走 dotenv，**键已存在就不覆盖**，而 `process.env` 里的空字符串也算「已存在」。
- 后果：构建期内联出 `""` → 请求打到前端自身同源 → 拿到 HTML 而非 JSON。
- 判别：找一个只在 `.env.production` 里定义、面板绝不会配的变量（本项目用 `NEXT_PUBLIC_ICONFONT_CDN`）去产物里查：
  - 它生效 ⇒ 文件已加载，只是那两个被空串覆盖；
  - 它也没生效 ⇒ 整个文件没被加载（转坑 2）。
  - 注意这类值可能在 **HTML**（layout 的 `<link>`）而不在 JS chunk 里，两处都要 grep。
- 已加固：`src/lib/api.ts` 内 `PROD_API_BASE` / `PROD_AI_BASE` 常量，env 为空时回落线上域名。换域名时改这两处 + `.env.production`。

### 坑 2：`.vercelignore` 的 `.env` / `.env.*` 把分层配置整个排除

Python 两端（ai-py / backend）的 `.vercelignore` 里有 `.env` + `.env.*`，会把 `.env.development` / `.env.production` **一起排除在上传之外**，线上文件根本不存在。

- 判别：AI `/health` 的 `retrieval_enabled` 生产应为 `false`，若返回 `true` ⇒ 文件没读到。
- ❌ 别用 `!` 否定：试过在后面加 `!.env.development` / `!.env.production`，**Vercel 侧不生效**（虽说是 gitignore 语法）。
- ✅ 正确修法：不写 `.env.*` 通配，改为精确列举含密钥的文件：

  ```
  .env
  .env.local
  .env.*.local
  ```

  `.env.development` / `.env.production` 天然不在排除之列。

- 前端没有 `.vercelignore`，不受影响。**新增任何配置文件后，都要确认它没被部署链路过滤掉。**

### 坑 3：改了面板环境变量不会自动触发部署

Vercel Git 集成只对 **push commit** 自动部署。只改面板变量 → 必须手动 Redeploy（或再 push 一个 commit）。
反过来，**push 了就不需要手动 Redeploy** —— 会自动构建（实测约 1 分钟）。

### 坑 4：生产判定真正生效后，缺 `ENCRYPTION_KEY` 会让后端整体崩溃

`app/core/crypto.py` 在**模块 import 期**构建 Fernet，生产判定为真而 `ENCRYPTION_KEY` 未配置时直接 `raise RuntimeError`
→ Vercel 表现为 `A server error has occurred` + `FUNCTION_INVOCATION_FAILED`（进程没起来，不是业务 500）。

- 之前"正常"只是因为 `.env.production` 没上传、`APP_ENV` 未设，一直以 development 判定在跑，安全校验被跳过。
- 本地复现（不用看 Vercel 日志）：`VERCEL=1 APP_ENV=production python -c "from app.main import app"`，会直接抛出缺哪个变量。
- 修复：面板配 `ENCRYPTION_KEY`，生成方式：

  ```bash
  python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
  ```

- ⚠️ 更换 key 会导致**已加密的存量数据无法解密**，换之前先确认库中是否已有加密数据。

### 提交/推送提醒

- **根 monorepo 的 push 不触发任何部署**（Vercel 项目连的是各自子仓）。改哪个服务就在哪个子仓提交：
  `oraclemind` / `oraclemind-backend` / `oraclemind-ai-py`。
- 三端 remote 已统一为 SSH，本机 `~/.ssh/id_ed25519` 可直接推；HTTPS remote 会因无落盘凭证报 `could not read Username`。
