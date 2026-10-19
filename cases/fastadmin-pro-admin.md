# Case Study: FastAdmin Pro — Vue3 + FastAPI Admin Template

A standard admin backend built on Vue3 + FastAPI that I use to show full-stack delivery, permission design, and deployment. It also ships a device-management module as a worked customization example, to prove the template turns into a real business backend.

> 🌐 中文版 / Chinese version: [fastadmin-pro-admin.zh.md](fastadmin-pro-admin.zh.md)

---

## Background

Most small factories and device makers want an internal admin backend, and fast. The common off-the-shelf templates are Java (RuoYi) and awkward to bend toward custom IoT workflows. I built this one in Python/FastAPI so it matches how I actually deliver, and so it doubles as a reusable base for back-office customization jobs.

---

## Problem

- No good Python/FastAPI admin starting point to show clients.
- Permission models in many templates are hardcoded, not data-driven.
- Audit trails are usually missing or added by hand in every endpoint.
- Bulk data (devices, users) has no clean import/export path.

---

## Constraints

| Constraint | Detail |
|-----------|--------|
| Stack fit | Vue3 + FastAPI, matching my delivery stack |
| Permission model | Data-driven RBAC: catalog / menu / button, not inline checks |
| Auditability | Operation log must be automatic, not per-endpoint |
| Demo safety | A public demo must not let visitors corrupt shared data |
| Maintainability | Clean split: api / models / schemas / services / deps |

---

## Solution

```
┌──────────┐  /api   ┌────────────┐  SQLAlchemy  ┌─────────┐
│ Vue3 SPA │────────►│  FastAPI   │────────────►│  MySQL  │
│ Element  │ JWT Bearer│ RBAC      │             └─────────┘
│  Plus    │◄────────│ Excel/Log  │  Redis       ┌─────────┐
└──────────┘  JSON   └────────────┘────────────►│  Redis  │
                          │ Nginx reverse proxy  └─────────┘
                          ▼
              docker-compose: mysql + redis + backend + frontend + nginx
```

The whole thing runs from one `docker compose up -d` — MySQL, Redis, the FastAPI backend, the Vue build served by Nginx, and the API behind it.

- **Backend layout**: `api / core / deps / models / schemas / services` plus `main.py`. Business logic stays in `services`, routes stay thin.
- **Auth**: PyJWT access + refresh tokens; bcrypt password hashing; an `admin` superuser seeded on first boot. Refresh tokens live in Redis so they can be revoked.
- **RBAC**: `sys_user / sys_role / sys_menu` with menu types `catalog / menu / button`. The frontend builds its routes from the menu API and shows or hides buttons by permission codes like `system:user:add`.
- **Operation log**: captured by a FastAPI dependency/middleware automatically — who, method, url, ip, module, time — so business endpoints don't log by hand.
- **Soft delete**: every base table carries a `deleted_at`; deletes just flag the row, queries filter it out by default.
- **Unified response**: one envelope for all business endpoints:

```json
{ "code": 200, "message": "success", "data": { "list": [], "total": 0, "page": 1, "page_size": 10 } }
```

- **Excel**: user and device bulk import/export with per-row error reporting.
- **The customization proof**: the device-management module is built on the same template — it shows the path from generic admin to a concrete business backend (device list, status, CRUD), which is the exact work most back-office customization posts ask for.

The full design spec (architecture, DB schema, permission model, API contract) is in the project's `ForAI.md`.

---

## Key Technical Decisions

### 1. Dual-token auth instead of one JWT
Short-lived access tokens limit the damage if one leaks; refresh tokens in Redis can be revoked and let the frontend refresh silently. It's a small change that reads as "done properly" to anyone who's shipped auth before.

### 2. Menu drives the frontend, not the other way around
Storing menus (catalog/menu/button) in a table means a non-developer can reshape permissions, and the frontend generates routes from one source. No duplicated route definitions to drift.

### 3. Operation log as middleware, not in every endpoint
Putting it in a dependency means zero business-code pollution and a consistent trail everywhere. Once it's set up, nobody thinks about it again.

### 4. Soft delete by default
A `deleted_at` on every base table avoids accidental data loss and keeps an audit history. Queries just filter it out.

---

## What I learned

The part that makes a template actually reusable for customization is the structure — deps → services → api, a real permission model, and a response contract — not any single feature. Get those right and a new business backend is assembly, not archaeology.

---

## Tags

`#full-stack` `#fastapi` `#vue3` `#rbac` `#iot-admin`
