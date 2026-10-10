# 合规整改记录

> 本文件记录玄镜（CN 左轨）为规避「封建迷信骗财」高危定性所做的产品与工程整改。出海产品（Xuanjing / 觉境）适用对应区域法规，不沿用本套话术。

## 1. 三要件模型（高危判据）

同时满足以下三点即构成「封建迷信骗财」高危：

1. **吉凶祸福结论**（预测吉凶、命运定论）
2. **消灾方法 / 物品**（解法、转运物、法器）
3. **获利 / 引流**（付费、导流、私域）

整改原则：**按拆要件排序**——先去吉凶定论话术，再去外链/获利引导。**单纯免责声明无效**，必须从内容层面拆解要件。

## 2. 已落地项（代码已确认）

| 整改项 | 状态 | 说明 |
|--------|------|------|
| 风水页整页下线 | ✅ | `/fengshui` 只剩迁移提示页；高危表零引用，`starData.ts` 已归档 `.archive/` |
| 阶段 2 文案改造（吉凶→觉察） | ✅ | `cemingData` 等，grep 命中 17+ 文件 |
| AI 合规护栏（prompt 层） | ✅ | `INJECTION_GUARD` / `COMPLIANCE_BLOCK` / `TONE_BLOCK` 全模块注入 |
| 输出层外链硬清洗 | ✅ | `home_agent._strip_external_links` / `_flush_link_safe`，流式也不漏 URL |
| PIPL 基建 | ✅ | `lib/piplConsent.ts` + `pipl/AssessmentConsentModal.tsx` |
| 支付去对价化 | ✅ | prompt 改造层，去掉「付费改命」对价表述 |

## 3. 措辞规范（用户可见文案）

- **禁用腔**：算命 / 排盘 / 占卜 / 命盘 / 命局 / 命格 / 命理 / 算一算。
- **可保留的产品自有词**：术数、本命盘、数字密码（「数字命理」全站统一为「数字密码」）。
- **统一框架**：觉察档案 / 解读记录 / 今日指引。
- **CTA 硬编码**：在 `home_agent.py` 的 `_CTA_TARGETS` / `_CTA_EXPLORE_LABELS`；`agent/prompts.py` 新增 5.1 措辞规范。
- **缓存版本**：`_prompt_fmt_hash` 只覆盖 `src/prompts/*` + formatter；改 `src/agent/*.py` 的 prompt 必须手动 bump `_CACHE_VER`（当前 v15）。

## 4. 产品级合规特性

- **首页带问题跳转**：`app/page.tsx` 的 `lastUserQ` + `buildNavHref()` —— 首页说过困扰后，宫格/头部入口自动带参：`/bugua?q=&autodiv=1`、`/tarot?q=&autostart=1`、`/horoscope`、`/numerology` 只带 `?q=`。诉求截断 60 字；过滤只排除闲逛意图，勿把「不知道」当无诉求。
- **复原力测评**：前端 `/assessment`（问卷→单独同意→付费闸门→SSE 报告），合规商品 `resilience_assess`（¥9.9，不在 PAUSED_ITEMS），AI `POST /api/v1/agent/assessment/stream`（`src/agent/assessment_agent.py`）。prompt 双端镜像：改 `oraclemind-ai` 必须同步 `lib/prompts/resilienceAssessment.ts`。
- **测评导流**：AI 遇测评类请求禁外链只推自家（`prompts` 5.2，v19）；CTA 新增 `scales → /scales`。

## 5. 排查口诀

「AI 表现异常」**先查是不是没重启 / 旧缓存**——改 AI 源码只改磁盘，常驻进程仍跑旧代码；`data/cache_store.json` 会回放旧响应。详见 `docs/ai-troubleshooting.md`。
