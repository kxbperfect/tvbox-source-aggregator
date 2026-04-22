# CLAUDE.md
This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands
- Dev (CF Worker): `npm run dev:cf`
- Deploy (CF Worker): `npm run deploy`
- Build (Node.js): `npm run build:node`
- Start (Node.js): `npm start` or `node scripts/start.js`
- Type check: `npx tsc --noEmit`

## Architecture
TVBox Source Aggregator — 将多个 TVBox 配置源合并成一个稳定的聚合地址。

### 双入口
- `src/cf-entry.ts` — Cloudflare Worker 入口（使用 KV 存储）
- `src/node-entry.ts` — Node.js 入口（使用 SQLite/JSON 文件存储，含 cron 定时任务）

### 核心模块
- `src/routes.ts` — HTTP 路由（Hono 框架），管理后台、配置输出、代理等所有端点
- `src/aggregator.ts` — 聚合流程编排，串联 fetch → parse → merge → dedup → speedtest

### 业务逻辑 (`src/core/`)
- `fetcher.ts` — 批量 fetch 配置 JSON，支持多仓递归展开
- `parser.ts` — 配置 JSON 规范化
- `merger.ts` — 站点级合并引擎
- `dedup.ts` — 站点去重逻辑
- `speedtest.ts` — zbape.com 测速 API 集成
- `decoder.ts` — 配置解码器（图片伪装 Base64 + AES CBC/ECB 加密）
- `jar-proxy.ts` — Spider JAR 代理（MD5 映射 + URL 转发）
- `live-source.ts` — 直播源聚合与测活
- `maccms.ts` — MacCMS 源验证与转换
- `blacklist.ts` — 站点黑名单管理
- `admin.ts` — 管理后台 HTML 页面
- `dashboard.ts` — 监控仪表盘页面
- `config-editor.ts` — 可视化配置编辑器
- `config.ts` — KV 键名常量和默认配置
- `types.ts` — TypeScript 类型定义
- `cleaner.ts` — 配置清理工具

### 存储抽象层 (`src/storage/`)
- `interface.ts` — Storage 接口定义
- `kv.ts` — Cloudflare KV 适配
- `sqlite.ts` — SQLite 实现（Node.js）
- `json-file.ts` — JSON 文件降级实现

### 部署
- **Cloudflare Worker**: `wrangler.toml` 配置，KV namespace 绑定
- **Docker**: `Dockerfile` + `docker-compose.yml`，发布到 Docker Hub `rio22/tvbox-aggregator`
- **一键脚本**: `start.command`(Mac) / `start.bat`(Win)

## Build shape
- TypeScript，Hono 框架
- CF Worker 入口由 wrangler 直接处理
- Node.js 入口由 `scripts/build.js` 通过 esbuild 编译到 `dist/`
- 无测试框架（当前无 test 目录）

## Key patterns
- 存储层通过 `Storage` 接口抽象，CF 用 KV，Node 用 SQLite，降级用 JSON 文件
- 聚合流程：fetch 多个源 → 解析规范化 → 合并去重 → 可选测速 → 输出 TVBox 配置 JSON
- 管理后台和仪表盘是内联 HTML（admin.ts / dashboard.ts），不依赖前端构建
- JAR 代理通过 MD5 映射表将相对路径转换为绝对 URL，支持 CF Worker 边缘代理
- 配置解码器支持图片伪装（提取 Base64 payload）和 AES 加密（CBC/ECB 模式）

## Repository-specific notes
- 两个远程仓库：origin (Gitee 对外分发) + github (GitHub 公开版)
- Docker 镜像通过 GitHub Actions 自动构建多架构版本（amd64/arm64/armv7）
- `.env` 包含敏感信息（ADMIN_TOKEN），不提交
- `data/` 目录存放运行时数据（SQLite db、缓存），不提交
- `wrangler.toml` 中的 KV namespace ID 是生产环境值，谨慎修改
