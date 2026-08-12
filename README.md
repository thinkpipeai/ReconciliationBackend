# Frontend-Backend Separation Refactoring — Architecture and API Contracts
> Version: v1.0

## 1. Current Status
| Layer | Technology / Solution | Critical Path |
|------|-------------|----------|
| Frontend | React 19 + Vite 8 + Tailwind CSS 4 | `src/` |
| Routing | React Router 7 | `src/main.jsx` |
| Corporate Website | Single-page static content, bilingual (Chinese and English) | `src/App.jsx` |
| Reconciliation Sub-app | `/reconcile/*` routing | `src/reconcile/` |
| Data Layer | Supabase JS client directly connected to PostgreSQL | `src/lib/reconcileApi.js` |
| Authentication | Plaintext password lookup table + localStorage session | `src/lib/auth.js` |
| Database | Supabase (3 tables) | `supabase/schema.sql` |
| Deployment | GitHub Actions → GitHub Pages | `.github/workflows/deploy.yml` |
| Public URL | https://thinkpipeai.tech | — |
### 1.1 Reconciliation Business Module
| Route | Role | Function |
|------|------|------|
| `/reconcile/login` | All | Login (admin / employee) |
| `/reconcile/admin` | admin | Today’s Summary, Employee Management, Today’s Records, Settlement |
| `/reconcile/employee` | employee | View/Add Today’s Service Records |
### 1.2 Existing Databases (Supabase / PostgreSQL)
| Table | Purpose |
|----|------|
| `employees` | Employees and administrators (username, password, role, commission_rate) |
| `records` | Service records (employee_id, date, service, payment, amount, tip) |
| `settlements` | Daily settlements (settlement_date, data JSONB) |
Seed account: `admin / admin` (role: admin)
### 1.3 Current Data Flow
React component → reconcileApi.js → @supabase/supabase-js → Supabase PostgREST → PostgreSQL

---
## 2. Target Architecture
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

## 3. Scope of Frontend Changes
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

## 4. REST API Contract
    4.1 API Overview
        login | POST | /api/auth/login | body: { “username”, “password” }
        checkEmployeesAccessible | GET | /api/health/employees | Returns whether employee data is available
		fetchEmployees | GET | /api/employees | role=employee only
        createEmployee | POST | /api/employees | Create an employee
        deleteEmployee | DELETE | /api/employees/{id} | Delete an employee
        fetchTodayRecords | GET | /api/records/today | Includes the employees.name association
		fetchEmployeeTodayRecords | GET | /api/records/today?employeeId={id} | Today’s records for a specific employee
        createRecord | POST | /api/records | Add a new service record
        fetchTodaySummary | GET | /api/summary/today | Backend aggregation
        fetchSettlementForToday | GET | /api/settlements/today | Today’s settlement
		fetchAllSettlements | GET | /api/settlements | List of historical settlements
        generateSettlement | POST | /api/settlements/generate | Settlement logic migrated from JS to Java
        Additional: GET /api/health → { “status”: “ok” }
    4.2 Field Naming Conventions
		Backend Java DTOs are written in camelCase; JSON serialization uses snake_case: Jackson’s global `PropertyNamingStrategies.SNAKE_CASE`
        Snake_case fields used by the frontend:
    4.3 Enum Constraints (Consistent with schema)
        role (admin, employee)
        service (Massage, Cupping, Acupuncture)
		payment (Cash, Check, Card)
    4.4 Authentication Conventions (Phase 1)
        Login: POST /api/auth/login; upon success, returns an Employee object (excluding the password)
        Session: Frontend continues to use localStorage (auth.js remains unchanged)
        Password Storage: Backend uses BCrypt
        Administrative API Protection: Optional request header X-User-Role: admin
