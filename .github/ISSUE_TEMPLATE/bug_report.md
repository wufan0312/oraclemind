---
name: 缺陷报告 / Bug Report
about: 报告一个可复现的问题
title: "[BUG] "
labels: bug
---

## 影响的服务
<!-- 前端 / 后端 / AI 服务 / 管理后台 / 小程序 / 主仓库文档 -->

## 现象描述
<!-- 期望是什么，实际是什么 -->

## 复现步骤
1.
2.
3.

## 环境
- 环境：生产（vercel.app）/ 本地（:3000/:8000/:8021）
- 浏览器 / 设备（如适用）：
- 账号 /  visitorId（如涉及）：

## 排查线索（AI 服务必填）
- [ ] 是否已重启 AI 服务（`scripts/ai_server.py restart`）
- [ ] 是否已清 `data/cache_store.json` 旧缓存
- [ ] 改 prompt 后是否 bump `_CACHE_VER`
- [ ] 线上是否 Redeploy

## 日志 / 截图
<!-- 粘贴相关控制台 / 服务端日志，或贴图 -->
