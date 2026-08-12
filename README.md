# 前后端分离改造 — 架构与 API 契约
> 版本：v1.0  
> 日期：2026-08-08
## 1. 现状
| 层级 | 技术 / 方案 | 关键路径 |
|------|-------------|----------|
| 前端 | React 19 + Vite 8 + Tailwind CSS 4 | `src/` |
| 路由 | React Router 7 | `src/main.jsx` |
| 企业官网 | 单页静态内容，中英双语 | `src/App.jsx` |
| 对账子应用 | `/reconcile/*` 路由 | `src/reconcile/` |
| 数据层 | Supabase JS 客户端直连 PostgreSQL | `src/lib/reconcileApi.js` |
| 认证 | 明文密码查表 + localStorage 会话 | `src/lib/auth.js` |
| 数据库 | Supabase（3 张表） | `supabase/schema.sql` |
| 部署 | GitHub Actions → GitHub Pages | `.github/workflows/deploy.yml` |
| 公网地址 | https://thinkpipeai.tech | — |
### 1.1 对账业务模块
| 路由 | 角色 | 功能 |
|------|------|------|
| `/reconcile/login` | 全部 | 登录（admin / employee） |
| `/reconcile/admin` | admin | 今日汇总、员工管理、今日记录、结算 |
| `/reconcile/employee` | employee | 查看/添加今日服务记录 |
### 1.2 现有数据表（Supabase / PostgreSQL）
| 表 | 用途 |
|----|------|
| `employees` | 员工与管理员（username, password, role, commission_rate） |
| `records` | 服务记录（employee_id, date, service, payment, amount, tip） |
| `settlements` | 日结算（settlement_date, data JSONB） |
种子账号：`admin / admin`（role: admin）
### 1.3 当前数据流
React 组件 → reconcileApi.js → @supabase/supabase-js → Supabase PostgREST → PostgreSQL


结算逻辑 `generateSettlement()` 目前在前端 JavaScript 中执行。
---
## 2. 目标架构
```mermaid
flowchart LR
  subgraph frontend [Frontend_GitHubPages]
    UI[React_UI_不变]
    ApiLayer[reconcileApi.js_改调用REST]
  end
  subgraph backend [Backend_云服务器]
    SB[Spring_Boot]
    MySQL[(MySQL)]
  end
  UI --> ApiLayer
  ApiLayer -->|"HTTPS /api/*"| SB
  SB --> MySQL

2.1 Data Flow After the Refactoring
    React component (unchanged) → reconcileApi.js (modified) → fetch REST → Spring Boot → MySQL

2.2 Deployment Plan
    Component            Deployment Location            Notes
    Frontend SPA        GitHub Pages        Domain thinkpipeai.tech, HTTPS
    Backend API        Cloud Server (VM)        Nginx reverse proxy; subdomain api.thinkpipeai.tech recommended
    Database            Cloud Server MySQL 8    Access via localhost only; not exposed to the public internet

2.3 Key Constraints
    1. GitHub Pages uses HTTPS; the backend API must also use HTTPS (to avoid mixed content)
    2. Frontend CORS must allow requests from https://thinkpipeai.tech and http://localhost:5173
    3. Data Migration: Only admin seed data is required; do not export historical data from Supabase

3. Scope of Frontend Changes
    No changes to any .jsx components under `src/reconcile/`; only modify data integration and environment configuration.
    Rewrite `src/lib/reconcileApi.js`: Replace Supabase calls with `fetch` requests to the REST API; keep the export function signatures unchanged
    Add `src/lib/httpClient.js` to standardize `baseURL`, JSON, and error handling
	Modify `.env.example`: Replace `VITE_SUPABASE_*` with `VITE_API_BASE_URL`
    Modify `vite.config.js`: Change the dev proxy from `/api` to `http://localhost:8080`
    Modify `.github/workflows/deploy.yml`: Inject `VITE_API_BASE_URL` into CI and remove Supabase secrets
	Modified `package.json`: Removed `@supabase/supabase-js`
    Deleted `src/lib/supabase.js` (retained until Day 8 for rollback purposes)
    `src/lib/auth.js` remains unchanged; continues to use `localStorage` for sessions
    `src/reconcile/**/*.jsx` remains unchanged; UI and business logic remain untouched
