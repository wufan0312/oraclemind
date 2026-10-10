# AI 服务排查手册

> 玄镜 AI 服务（`oraclemind-ai`）的「表现异常」绝大多数不是代码写错，而是**没重启 / 旧缓存回放**。排查顺序固定如下。

## 1. 第一反应：先查重启与缓存

改 AI 源码**只改了磁盘**，常驻进程仍跑旧代码；且 `data/cache_store.json` 会回放旧响应。所以：

> 「AI 表现异常」≠「代码有问题」，先怀疑「没重启 + 旧缓存」。

## 2. 诊断：对比缓存内的 prompt 版本

用诊断脚本取缓存 key 内的 prompt hash，与当前 `_home_prompt_hash()` 对比：

- 若缓存里的 `agent_action = 旧版本` → 证明是**旧缓存回放**，不是新代码生效。

## 3. 处置

```bash
# 重启 AI 服务（内部自切 .venv）
cd oraclemind-ai && python scripts/ai_server.py restart
# 清对应缓存（删 data/cache_store.json 中相关条目，或整文件清空后重启）
```

- **线上**：Vercel 需重新部署（Redeploy）才会跑新代码；只改面板变量也要 Redeploy。

## 4. 改 prompt 必看：缓存版本号

- `_prompt_fmt_hash` 只覆盖 `src/prompts/*` + formatter。
- 改 `src/agent/*.py` 里的 prompt 文案时，**必须手动 bump `_CACHE_VER`**（当前 `v15`），否则旧缓存继续命中。

## 5. 外链硬清洗（输出层）

- `home_agent._strip_external_links` / `_flush_link_safe`：在 prompt 层（`INJECTION_GUARD` / `COMPLIANCE_BLOCK` / `TONE_BLOCK`）之外，于输出层再洗一遍，**流式输出也不漏 URL**。
- 若发现 AI 仍吐出外链，先确认：① 是否重启；② `_CACHE_VER` 是否 bump；③ 是否清了缓存。

## 6. 流重复 / 孤立编号标题

GLM-4-Flash 已知缺陷（stream 重复、孤立编号标题），缓解由共享 `src/lib/textDedup.ts`（5 fns，含 `renumberStandaloneNumberedHeadings`）pipeline 覆盖；`src/lib/markdown.ts` 的 `mdToHtml` 跨页面复用。

## 7. 本地探活

```bash
curl -s https://oraclemind-ai-eight.vercel.app/health   # llm_available: true 才正常
curl -sI -H 'Origin: https://oraclemind-frontend.vercel.app' \
     https://oraclemind-ai-eight.vercel.app/health | grep -i access-control-allow-origin
```

> 判错口诀见 `docs/deployment.md` §6：`FUNCTION_INVOCATION_FAILED` = 进程没起来；JSON 带 `detail`/`error` = 应用活着。
