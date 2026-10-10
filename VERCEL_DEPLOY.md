# 玄镜 OracleMind · Vercel 部署总手册（CLI 直连版）

> 适用场景：**github.com 网络不通**（本机实测 2026-09-14），无法用 Vercel 的 Git 集成，
> 全流程改用 **Vercel CLI 从本地直接部署**（vercel.com 可达）。
>
> 三个子服务各自作为**独立 Vercel 项目**部署，互不共享配置：

| Vercel 项目 | 本地目录 | 形态 | 域名（示例） |
|---|---|---|---|
| oraclemind-frontend | `oraclemind/` | Next.js 14 | `https://oraclemind-frontend.vercel.app` |
| oraclemind-backend | `oraclemind-backend/` | Python ASGI (FastAPI) | `https://oraclemind-backend.vercel.app` |
| oraclemind-ai-eight | `oraclemind-ai-py/` | Python ASGI (FastAPI) | `https://oraclemind-ai-eight.vercel.app` |

外部依赖（后端/AI 必需）：
- **Neon**（免费托管 PostgreSQL）→ 后端 `DATABASE_URL`（Vercel 上禁 SQLite，有代码防呆）
- **Upstash**（免费托管 Redis，TLS）→ 两个 Python 服务的缓存

**部署顺序：后端 → AI → 前端**（前端要在构建时拿到两个服务的域名注入 `NEXT_PUBLIC_*`）。

> 📄 **环境变量怎么配，请看 [`ENV_SETUP.md`](./ENV_SETUP.md)**。
> 三端已统一为「按环境自动切换」：本地用 `.env.development`（localhost），
> 生产用 `.env.production`（线上域名），两个文件都随仓库提交；
> 密钥一律走 Vercel 面板或本地 `.env`。本文只讲部署动作，不再重复变量清单。

---

## 0. 前置准备（一次性）

### 0.1 安装并登录 Vercel CLI

```bash
npm i -g vercel        # 或全程用 npx vercel
vercel login           # 浏览器完成授权；无法弹浏览器时用 token：
                       # vercel.com → Settings → Tokens → 建 token → setx VERCEL_TOKEN xxx
```

### 0.2 注册 Neon（拿 PG 连接串）

1. https://neon.com 注册（免费）→ 建项目（region 选 Singapore，离国内最近）
2. Dashboard → Connection string → 选 **Pooled connection**（带 `-pooler`）
3. 形如：`postgres://user:pass@ep-xxx-pooler.ap-southeast-1.aws.neon.tech/neondb?sslmode=require`
4. **改前缀**为 Vercel 用格式：`postgresql+asyncpg://user:pass@.../neondb?sslmode=require`
   （只改 `postgres://` → `postgresql+asyncpg://`，其余原样）

### 0.3 注册 Upstash（拿 Redis 连接串）

1. https://upstash.com 注册（免费）→ Create Database（region 同样建议新加坡）
2. 拿到 `rediss://xxx:yyy@xxx.upstash.io:6379/0`（**注意是 rediss**，Vercel 走 TLS）
3. Upstash 免费版 Endpoint 格式若是 `redis://` 需手动加 `s`

> 密钥已备好：`oraclemind-backend/.env.prod`（gitignored）内含已生成的
> `JWT_SECRET` / `ENCRYPTION_KEY`，以及待替换的 Neon/Upstash 占位符。AI 服务的
> `AI_API_KEY`（智谱）复用 `oraclemind-ai-py/.env` 里现有的值。

---

## 1. 部署后端（oraclemind-backend）

```bash
cd oraclemind-backend

# 首次：关联/创建项目（交互里选 Create new project，名字建议 oraclemind-backend）
vercel link

# 逐条添加环境变量（production 环境；值从 .env.prod 复制，DATABASE_URL/REDIS_URL 换成 0.2/0.3 的真实串）
vercel env add APP_ENV production          # 值: production
vercel env add DEBUG production            # 值: false
vercel env add DATABASE_URL production     # 值: postgresql+asyncpg://...
vercel env add REDIS_URL production        # 值: rediss://...
vercel env add JWT_SECRET production
vercel env add ENCRYPTION_KEY production
vercel env add CORS_ORIGINS production     # 暂填前端预估域名 https://oraclemind-frontend.vercel.app

# 部署到生产
vercel deploy --prod
```

说明：
- `vercel.json` 已配 `buildCommand: "alembic upgrade head"` —— 构建时对 Neon 自动建表（幂等）。
  若 build 阶段报数据库连接错误，先检查 `DATABASE_URL` 前缀是否为 `postgresql+asyncpg://`。
- CORS 域名等前端部署完拿到真实域名后，回来改：`vercel env rm CORS_ORIGINS production` 再 add，然后重新 `vercel deploy --prod`。

**验证**：`curl https://<backend>.vercel.app/api/health` 应返回 200 且 `database: up / redis: up`。

---

## 2. 部署 AI 服务（oraclemind-ai-py）

```bash
cd oraclemind-ai-py

vercel link                # 新建项目，名字建议 oraclemind-ai

vercel env add AI_PROVIDER production      # zhipu
vercel env add AI_MODEL_CHAT production    # glm-4-flash
vercel env add AI_MODEL_LITE production    # glm-4-flash
vercel env add AI_API_KEY production       # 复用 oraclemind-ai-py/.env 里的智谱 key
vercel env add AI_BASE_URL production      # https://open.bigmodel.cn/api/paas/v4
vercel env add AI_TOTAL_TIMEOUT_MS production  # 90000
vercel env add APP_ENV production          # production
vercel env add DISABLE_CACHE production    # 0
vercel env add RETRIEVAL_ENABLED production    # true（保留 RAG，见下方体积说明）
vercel env add CHROMA_PERSIST_DIR production   # /tmp/oraclemind_chroma（Vercel 只读盘，必须指 /tmp）
vercel env add REDIS_URL production        # rediss://...（Upstash，同后端）
vercel env add CORS_ORIGINS production     # https://<frontend>.vercel.app

vercel deploy --prod
```

**验证**：`curl https://<ai>.vercel.app/health` 应 200 且 `llm_available: true`。

### ⚠️ chromadb 体积风险（保留 RAG 的已知代价）

requirements.txt 含 chromadb，其传递依赖（kubernetes/onnxruntime/numpy 等）合计数百 MB，
逼近 Vercel 500MB 函数包体上限。**若部署失败（报 size limit / bundle too large）**，按 Plan B 降级：

1. `requirements.txt` 临时删掉 `chromadb>=0.5.0` 一行（代码是惰性导入，`src/services/retrieval.py:61`，不装包不报错）；
2. `vercel env rm RETRIEVAL_ENABLED production` 后重新 add 为 `false`；
3. `vercel deploy --prod`。解读功能退化为 LLM + 本地规则（RAG 检索增强关闭）。
4. 本地 venv 已装 chromadb，本地开发不受影响。

---

## 3. 部署前端（oraclemind/）

```bash
cd oraclemind

vercel link                # 新建项目，名字建议 oraclemind-frontend

# ⚠️ NEXT_PUBLIC_* 是构建时注入，必须先加 env 再 deploy，且改值必须重新部署
vercel env add NEXT_PUBLIC_API_BASE production      # https://<backend>.vercel.app（不含 /api！）
vercel env add NEXT_PUBLIC_AI_API_BASE production   # https://<ai>.vercel.app（不含 /api！）
vercel env add NEXT_PUBLIC_ICONFONT_CDN production  # https://at.alicdn.com/t/c/font_5225625_q91x1qydrn.css

vercel deploy --prod
```

`vercel.json` 已配 `buildCommand: "npx next build"`（绕开本地 Windows 定制 wrapper）。

**验证**：打开 `https://<frontend>.vercel.app`，起一次卜卦排盘 + 一次 AI 解读，
DevTools Network 确认请求打到 `<backend>/api/v1/...` 与 `<ai>/api/v1/...` 且无 CORS 报错。

---

## 4. 交叉配置总表（CORS / API 地址）

| 项目 | 要配的变量 | 指向 |
|---|---|---|
| backend | `CORS_ORIGINS` | `https://<frontend>.vercel.app` |
| ai | `CORS_ORIGINS` | `https://<frontend>.vercel.app` |
| frontend | `NEXT_PUBLIC_API_BASE` | `https://<backend>.vercel.app` |
| frontend | `NEXT_PUBLIC_AI_API_BASE` | `https://<ai>.vercel.app` |

规则：env 一律填**域名根路径，不含 `/api`**（`/api/v1/...` 由前端请求路径携带，写了就会双 `/api`）。

---

## 5. 已知限制与注意事项

- **Hobby 计划超时**：函数 `maxDuration=60` 在 Fluid compute 下可用；若实际跑 10s 被掐（排盘/AI 长请求偶发），升级 Pro。AI 服务流式解读（SSE）依赖流式响应，Vercel Python 函数已支持，但冷启动首包会慢几秒。
- **前端 no-store 头**：`next.config.js` 对所有页面路由设了 no-store（占卜结果实时性），只影响 CDN 缓存命中率，不影响功能。
- **本地 SQLite 数据不会自动迁移到 Neon**：如需搬历史数据，见 `oraclemind-backend/DEPLOYMENT.md` 第 5 节。
- **后续更新部署**：改完代码 `cd <目录> && vercel deploy --prod` 即可；改环境变量后同样要重新 deploy（前端尤其必须，NEXT_PUBLIC_* 构建时固化）。
- **admin 后台（oraclemind-admin）不在本次范围**，如需上线另配两个项目（frontend 3002 + backend 8001）。

---

## 6. 本次已落地的配置文件清单

| 文件 | 变更 |
|---|---|
| `oraclemind/vercel.json` | 新增：显式 `buildCommand: npx next build` |
| `oraclemind-backend/vercel.json` | 追加 `buildCommand: alembic upgrade head`（构建期自动建表） |
| `oraclemind-backend/.vercelignore` | 新增：排除清单（从 vercel.json 顶层 `excludeFiles` 迁入，见 6.1） |
| `oraclemind-ai-py/.vercelignore` | 新增：同上 |
| `oraclemind-backend/.env.prod` | 新增：生产密钥模板（JWT/ENCRYPTION 已生成真实值，gitignored） |
| `VERCEL_DEPLOY.md` | 本手册 |
| 后端/AI 的 `api/index.py`、`requirements.txt`、`pyproject.toml [tool.vercel]` | 此前已就绪，未改动 |

### 6.1 ⚠️ 顶层 `excludeFiles` 非法（2026-09-14 已修）

**现象**：GitHub 导入新建项目页红条报错 ——
`Invalid request: should NOT have additional property 'excludeFiles'. Please remove it.`
（错误发生在「建项目」这一步，还没到构建，所以与 Python 代码无关。）

**根因**：Vercel 官方 schema（https://openapi.vercel.sh/vercel.json）**顶层是 `additionalProperties: false`**，
允许的顶层键仅 `$schema / build / builds / cleanUrls / env / functions / git / github / headers /
images / name / redirects / rewrites / routes / regions / alias` 等，**没有 `excludeFiles`**。
`excludeFiles` 与 `includeFiles` 只允许写在 `functions.<entry>` 内部，且类型是**单个 glob 字符串**
（maxLength 256），不是数组。

**修法**：把排除规则从 `vercel.json` 顶层搬到同目录的 **`.vercelignore`**（语法同 `.gitignore`，
Vercel 上传/打包阶段直接跳过），`vercel.json` 只保留 `functions` 段。

**顺带清理**：AI 服务的 `build.env.PYTHON_VERSION` 属 deprecated 字段，已删除 —— Python 版本由
`functions."api/index.py".runtime = "python3.12"` 显式固定，效果等价。

> 三个子项目共用同一套规则：**`vercel.json` 只写 schema 允许的键；文件排除一律用 `.vercelignore`。**

### 6.2 导入时 `Project "xxx" already exists`

修掉 6.1 之后，导入页会转报重名：

```
Project "oraclemind-ai" already exists, please use a new name.
```

这不是配置错误。Vercel 的**项目名在同一 scope（团队/个人账号）内唯一**，而 `<项目名>.vercel.app` 域名又要求全局可用。
`oraclemind-ai` 这个名字此前被占用过（多为早先 CLI `vercel link` 建的空壳项目，或上一次点 Deploy 已把项目建出来）。

三选一：

| 做法 | 适用 | 代价 |
|---|---|---|
| **A. 删空壳后同名重建** | 该项目里没有手配的环境变量、也没部署过 | Project Settings 最底部 Delete Project（**会连带删掉该项目的 env**），再回导入页用原名 |
| **B. 换个名字导入**（最快） | 找不到/打不开那个旧项目 | 域名变为 `https://<新名>.vercel.app`，需同步 §4 交叉表里的 `CORS_ORIGINS` 与 `NEXT_PUBLIC_AI_API_BASE` |
| **C. 在旧项目上绑 Git** | 旧项目里已配好 env，想留着 | 项目 → Settings → Git → Connect Git Repository，Root Directory 填 `./` |

> 定位旧项目：Vercel 左上角 scope 切换器确认当前是 `WuFan` 团队还是个人账号 —— 项目只会出现在它所属的那个 scope 列表里。
> 或用 CLI 一把梭：`npx vercel link` 会把当前 scope 下**所有**项目列出来，选到已有的那个，再 `vercel deploy --prod`，不经过 Git 导入页。

---

## 6.3 报错：`Function Runtimes must have a valid version, for example \`now-php@1.0.0\``

又是 `vercel.json` 的 `functions` 段，这次是 `runtime` 字段的写法问题。

**根因**：Vercel 现在的校验器不接受 `functions.<entry>.runtime` 写成裸语言版本号（如 `"python3.12"`）。该字段的合法取值是「npm 包名@版本」格式（如社区运行时的 `vercel-php@0.5.2`，或官方 `@vercel/python@x.y.z`），
报错里那句 `now-php@1.0.0` 是老 `builds` 语法的遗留模板，说明校验走的是旧路径。
而官方内置运行时（Node/Python 等）其实**无需手写 runtime** —— 把 `.py` 放进 `api/` 目录，Vercel 会自动识别为 Python 函数。

**修法**：直接删掉 `functions.api/index.py` 里的 `"runtime"` 那一行即可，`maxDuration` / `memory` 保留。
最终形态：

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "functions": {
    "api/index.py": {
      "maxDuration": 60,
      "memory": 1024
    }
  }
}
```

> 后端 `oraclemind-backend/vercel.json` 本就未写 `runtime`（同样靠自动检测），两处保持一致。
> 若部署后想固定 Python 小版本，待项目成功创建后再在面板/CLI 里加 `"runtime": "python3.12"` 试一次（当前导入校验器会拦，故先省略）。

---

## 6.4 报错：`"tool.vercel.entrypoint" ... "api/index:app" but no matching module file was found`

第 3 关（runtime）之后的又一道 `vercel.json` / `pyproject` 校验关，这次是 **`pyproject.toml` 的 `[tool.vercel] entrypoint`**。

**根因**：`[tool.vercel] entrypoint = "api/index:app"` 让 Vercel 把 `api/index` 当成「可导入的 Python 模块」去解析，但构建期 `api/` 不是被当作包（无 `__init__.py`、也不在 import path 上），于是报 "no matching module file was found"。
Vercel 的提示已经给出解法：**删除该设置，改用自动检测**（auto entrypoint detection）。

**修法**：删掉 `oraclemind-ai-py/pyproject.toml` 与 `oraclemind-backend/pyproject.toml` 里的 `[tool.vercel] entrypoint = "api/index:app"` 整行。
`api/index.py` 顶层有 `from src.server import app` + `__all__ = ["app"]`，Vercel 会自动检测 `api/*.py` 为 Python(ASGI) 函数并加载其中的 `app` 对象，无需手写 entrypoint。

> 若将来确实要显式指定，格式须是「module:object」且模块文件真实可导入（如根目录 `main.py` 的 `main:app`），`api/index:app` 这种带目录前缀的写法在当前校验器下不认。

---

## 6.5 报错：`This Serverless Function has crashed` / `Code: FUNCTION_INVOCATION_FAILED`

**这一关和前四关性质完全不同：配置已经全对了。** 部署是**成功**的（构建通过、函数已上线），
挂的是**运行时** —— 函数进程在冷启动时就崩了，根本没走到 FastAPI。

### 先学会区分两种 500

| 你看到的页面 | 含义 | 排查方向 |
|---|---|---|
| Vercel 的「This Serverless Function has crashed」+ `FUNCTION_INVOCATION_FAILED` | 函数**进程级崩溃**，多数发生在 **import / 冷启动**阶段 | 依赖缺失、模块级代码报错、包结构被破坏 |
| 页面直接显示 `{"error":"SERVER_ERROR",...}` JSON | FastAPI **正常返回**的 500，全局异常处理器兜的底 | 业务代码异常，看 Function Logs 里的 `未处理异常` |

判据：后者是应用自己产出的响应，前者连应用都没起来。

### 本次根因（两类，都是「本地能跑、线上必崩」）

**根因 1 · 依赖从未被声明**

- `src/services/astrology.py:25` 是**模块级** `from pymeeus.Epoch import Epoch`
- `src/agent/report_agent.py:21` 是**模块级** `import aiohttp`
- 而这两个包**从未出现在** `requirements.txt` / `pyproject.toml` 中。

冷启动链：`api/index.py` → `src.server` → `src.routes.astrology` → `src.services.astrology`
→ `ModuleNotFoundError: No module named 'pymeeus'` → 函数崩溃。

本地之所以看不出来，是因为开发机的环境里早就手装过这两个包；而 Vercel 只按
`requirements.txt` 装依赖 —— **「本地能 import」不等于「线上能 import」**。

**根因 2 · `.vercelignore` 里 `_*.py` 规则连坐 `__init__.py`**

原先想用 `_*.py` 排掉临时脚本，但 gitignore 语义下 `_*` 会**连 `__init__.py` 一起匹配**
（`_*` 匹配 `__init__`）。后果：`src/` 下 **7 个 `__init__.py` 全部没被上传**，
子包退化成 namespace package，包内初始化逻辑静默失效。
`oraclemind-backend` 犯了**一模一样**的错（被排 9 个 `__init__.py`），已同步修。

> 临时脚本已统一放 `scripts/`（`scripts/` 整目录已被排除），`_*.py` 这条规则是多余的。

### 定位方法（三步，可复用）

1. **看错误形态** —— 按上表区分「函数崩溃」还是「应用返回 500」。这是最重要的一步。
2. **静态审计** —— 跑部署前自查脚本，一次性列出「源码 import 了但没声明」的包：
   ```bash
   python scripts/check_deps.py .                       # ai-py
   python scripts/check_deps.py ../oraclemind-backend   # backend
   ```
   同时它会模拟 `.vercelignore` 匹配，报出被误排除的源码文件。
3. **本地复现** —— 用**一个没装该包**的解释器跑一次入口，等价于 Vercel 的冷启动：
   ```bash
   python -c "from src.server import app; print('IMPORT OK')"
   ```
   缺包时会直接打印出 `ModuleNotFoundError` 与**具体文件行号**，与线上 500 完全对应。

### 修法

`oraclemind-ai-py/requirements.txt` 与 `pyproject.toml` 补上：

```
aiohttp>=3.9.0     # report_agent 并行回调排盘服务
PyMeeus>=0.5.12    # 占星本命盘实算（纯 Python，无需编译）
```

两个 `.vercelignore`（ai-py + backend）删掉 `_*.py` 那条规则，并留注释说明为什么不能写。

### 验证结果（本地，2026-09-14）

在隔离环境按 requirements 装齐依赖后：

```
IMPORT OK -> 玄镜 OracleMind · AI 服务      # 9 条路由全部注册成功
GET /health -> 200
{"status":"ok",...,"retrieval_enabled":true,"cache_backend":"memory(ttl)[redis-down]"}
```

Redis 不可用时按设计优雅降级到内存兜底，不影响可用性。

### 部署前必跑

```bash
python scripts/check_deps.py .
```

绿了再推。这一条能拦住 6.5 这类「构建成功但一访问就崩」的问题 —— 也是本次唯一一个
**不是配置问题**的坑。

---

## 6.6 现象：部署成功页显示 `{"detail":"Not Found"}` —— 这不是报错

Vercel 部署成功后，「Congratulations!」页面会自动请求部署 URL 的根路径并展示响应体。
如果你看到的是一个灰底 JSON：

```json
{"detail": "Not Found"}
```

**这代表部署成功、函数正常运行。** 它是 FastAPI / Starlette 的**默认 404 响应**，
说明应用已经起来了、只是没人为 `/` 定义路由。

### 怎么和「函数崩溃」区分

| 你看到的 | 谁产生的 | 结论 |
|---|---|---|
| `{"detail":"Not Found"}` | **FastAPI 自己**（`HTTPException` 默认 handler） | 服务正常，只是该路径无路由 |
| `{"error":"SERVER_ERROR","message":...}` | 本项目的全局异常处理器 | 应用活着，业务代码抛异常（看 Function Logs） |
| `This Serverless Function has crashed` / `FUNCTION_INVOCATION_FAILED` | **Vercel 平台** | 函数进程没起来（见 6.5） |
| `404: NOT_FOUND` + `Code: DEPLOYMENT_NOT_FOUND` | **Vercel 边缘网络** | 这个域名下没有部署（项目名/域名不对） |

一句话判据：**只要响应体是 JSON 且带 `detail` 或 `error` 字段，就说明你的应用在跑。**

### 本次处理

`oraclemind-ai-py/src/server.py` 增加根路由 `GET /`，返回服务名 / 版本 / 就绪状态，
并加入 `_RATE_LIMIT_WHITELIST`、`_AUTH_WHITELIST`（与 `/health` 同等待遇）：

```bash
curl https://<你的项目>.vercel.app/         # -> 200 {"status":"ok",...}
curl https://<你的项目>.vercel.app/health   # -> 200 含 retrieval_enabled / cache_backend
```

> 排查提示：`*.vercel.app` 的域名必须与**项目名**一致。若 `DEPLOYMENT_NOT_FOUND`，
> 说明你访问的域名没有部署 —— 去 Vercel 项目的 Domains 页复制真实域名。

---

## 7. 后端部署前置：不配这些，导入成功也会崩

后端比 AI 服务多一层「自我保护」：`app/db/init_db.py` 在 lifespan 里先调
`settings.validate_for_runtime()`，而**这个调用在 try 之外** —— 触发条件
`(is_production or is_vercel) and is_sqlite` 会直接 `raise RuntimeError` 阻止启动。

`is_vercel` 的判据是环境变量 `VERCEL=1`（Vercel 平台自动注入），所以在 Vercel 上
**只要 `DATABASE_URL` 还是默认的 SQLite，函数就是冷启动即崩**（页面表现与 6.5 完全一样：
`FUNCTION_INVOCATION_FAILED`）。这是刻意设计 —— SQLite 在函数实例上每次部署/冷启动都会丢数据。

### 7.1 必配环境变量（Vercel 项目 → Settings → Environment Variables）

| 变量 | 必填 | 说明 |
|---|---|---|
| `DATABASE_URL` | ✅ **必填** | 云 PostgreSQL，且**必须**是 `postgresql+asyncpg://` 驱动 |
| `JWT_SECRET` | ✅ 生产必填 | `APP_ENV=production` 时空值或默认 `change-me-in-production` 一律拒绝启动 |
| `ENCRYPTION_KEY` | ✅ | 敏感字段加解密；换值会导致旧密文不可解 |
| `CORS_ORIGINS` | ✅ | 逗号分隔的前端域名，如 `https://oraclemind-frontend.vercel.app` |
| `APP_ENV` | 建议 | 设 `production`（关闭 /docs、Cookie 强制 secure、日志级别收敛） |
| `REDIS_URL` | 可选 | 不配则降级为内存兜底（多实例语义不同，但不会崩） |
| `VERCEL` | 自动 | 平台注入，不要手动设置 |

### 7.2 `DATABASE_URL` 的正确写法（Neon 为例）

```
postgresql+asyncpg://<user>:<password>@<endpoint>-pooler.<region>.aws.neon.tech/neondb?ssl=require&prepared_statement_cache_size=0
```

三个必须注意的点：

1. **驱动**必须是 `postgresql+asyncpg://`，不能是 `postgresql://` 或 `postgres://`。
2. **SSL 参数是 `ssl=require`，不是 `sslmode=require`。**
   `sslmode` 是 libpq/psycopg2 的写法，**asyncpg 不认这个参数**，会抛
   `TypeError: connect() got an unexpected keyword argument 'sslmode'`。
3. **用 pooler 主机（结尾带 `-pooler`）时必须加 `prepared_statement_cache_size=0`。**
   PgBouncer 事务模式下服务端预处理语句会跨连接串味，不加会间歇性报
   `prepared statement "__asyncpg_stmt_x__" does not exist`。

> 本仓库 `oraclemind-backend/.env.prod` 里的模板行已按上述三点修正过，直接替换 `__NEON_*` 占位符即可。
> 迁移脚本 `migrations/env.py` 会把 `postgresql+asyncpg` 自动换成 `postgresql+psycopg2`（同步驱动）跑迁移，
> 所以 `psycopg2-binary` 必须在依赖里 —— 已在 requirements.txt 中。

### 7.3 构建期 `alembic upgrade head` 的取舍

`vercel.json` 里 `buildCommand: "alembic upgrade head"` 会在**构建阶段**连库建表。
好处是表在部署时就绪；代价是**构建会依赖数据库可达** —— 库地址错/密码错/Neon 被挂起时，
表现为「构建失败」而不是「运行时报错」。

应用 lifespan 里同样会跑一次迁移（`init_db()`，失败仅告警不阻断），所以：
**若希望构建不依赖数据库，把 `buildCommand` 整行删掉即可**，改由冷启动兜底建表（幂等）。

### 7.4 推送顺序（本机无落盘凭证时）

```bash
# 后端仓（独立 .git）
cd oraclemind-backend && git -c credential.helper= push origin main
# AI 仓（独立 .git）
cd oraclemind-ai-py && git -c credential.helper= push origin main
```

`-c credential.helper=` 临时清空凭证助手，让 git 在终端里直接提示输入
（用户名 = GitHub 账号，密码 = Personal Access Token）。
注意：**在仓库根目录 commit 不会进子仓**，四个子项目各有独立 `.git`（admin 除外）。

---

## 7.5 构建失败：`ValidationError: 2 validation errors for Settings`（debug / jwt_expire_minutes）

**这条不是 Vercel 的问题，也不是配置格式问题 —— 是环境变量的「值」被污染了。**

```
File "/vercel/path0/migrations/env.py", line 13, in <module>
  from app.core.config import settings
  ...
pydantic_core._pydantic_core.ValidationError: 2 validation errors for Settings
debug
  Input should be a valid boolean, unable to interpret input [type=bool_parsing]
jwt_expire_minutes
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing]
Error: Command "alembic upgrade head" exited with 1
```

注意它是**构建期**炸的：`buildCommand: alembic upgrade head` 会 import `migrations/env.py`
→ `app.core.config` → 实例化 `Settings()`，环境变量在这一步就被校验了。

### 根因：值被成对引号包住

Vercel 面板里 `DEBUG` 与 `JWT_EXPIRE_MINUTES` 的值实际是 `"false"` / `"10080"`（**带引号**）。
pydantic 的 bool / int 解析**不接受引号，也不接受首尾空白**。

**「恰好 2 个错误」是决定性线索**：只有「引号」这种脏值会同时让 bool 与 int 都失败。

| 喂进去的原始值 | 结果 |
|---|---|
| `"false"` / `"10080"`（成对引号） | ❌ **2 个错误**：`bool_parsing` + `int_parsing` ← 线上形态 |
| `false\r` / `10080\r`（CRLF 残留） | ❌ 1 个错误（int 能容忍 `\r`，bool 不能） |
| `false ` / ` 10080 `（首尾空格） | ❌ 1 个错误（只有 bool） |
| `false` / `10080` | ✅ 通过 |

复现命令（本地一条即可验证）：

```bash
cd oraclemind-backend
DEBUG='"false"' JWT_EXPIRE_MINUTES='"10080"' python -c \
  "from app.core.config import Settings; Settings()"     # 报同样的 2 个错
```

### 比「启动失败」更危险的：字符串字段的引号会静默改变语义

bool / int 字段会**大声报错**，字符串字段则会**悄悄带着引号用**：

| 变量 | 带引号时的后果 |
|---|---|
| `APP_ENV="production"` | `is_production` 判 **False** → 跳过 JWT_SECRET 强度校验 + 跳过 SQLite 拦截（**安全语义被绕过，不报任何错**） |
| `DATABASE_URL="postgresql+asyncpg://..."` | SQLAlchemy 解析 URL 失败 |
| `CORS_ORIGINS="https://a.vercel.app"` | 白名单带引号 → 浏览器预检全部失败 |

### 修法（已落地，`app/core/config.py`）

在**校验之前**统一清洗，让脏输入自动变成正确输入：

```python
@field_validator("*", mode="before")
@classmethod
def _normalize_env_scalars(cls, v):        # 所有字段：去首尾空白(含 \r) + 去成对引号
    return _clean_env_value(v)

@field_validator("debug", "proxy_trusted", mode="before")
...  # 布尔：支持 true/false/1/0/yes/no/on/off；非法值给出「哪个变量错了」的报错

@field_validator("port", "jwt_expire_minutes", ... , mode="before")
...  # 整数：容忍引号/空白/行内注释；空值回落默认值；非法值明确报错
```

洁癖无损：这些配置项本身不会包含首尾空格或引号，清洗只会把脏值修回正确值。
清洗后 `APP_ENV="production"` 能正确判为生产，安全校验恢复正常（已实测）。

### 验证（2026-09-14，本地隔离环境）

```
[OK ] 带成对引号（= 线上实际形态）  -> debug=False jwt_expire=10080
[OK ] 带 CRLF 残留                 -> debug=False jwt_expire=10080
[OK ] 大写 + 行内注释               -> debug=True  jwt_expire=10080
[OK ] yes + 空值回落默认            -> debug=True  jwt_expire=10080
[OK ] 真·非法值 maybe/abc          -> 报错信息直接点出 DEBUG / JWT_EXPIRE_MINUTES
IMPORT OK -> 玄镜 OracleMind 后端 | 路由数: 2
is_production: True | is_sqlite: False | debug: False | port: 8000
```

`alembic upgrade head` 在脏值环境下已能跑过配置关并开始执行迁移。

> 顺带发现（**与本次无关，SQLite 专属**）：某条数据迁移插入 `community_circles` 时用了
> PG 的 `now()`，SQLite 下报 `no such function: now` —— 已用干净 env 做对照组复现同样报错。
> 生产目标是 PostgreSQL，不受影响。

### 建议的环境变量填写姿势

1. **不要带引号**，Vercel 面板里直接填 `false`、`10080`、`production`（引号是 shell/JSON 的语法，不是给 env 的）。
2. **不要带行内注释**（`10080  # 7 天` 里的 `# 7 天` 会被算进值里）。
3. 避免从 Windows 记事本复制粘贴（尾部可能带 `\r`）。
4. 批量粘贴用 Vercel 的 **Paste .env** 功能，比逐条手填更不易出错。
