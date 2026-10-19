# 案例研究：FastAdmin Pro — Vue3 + FastAPI 后台模板

一个基于 Vue3 + FastAPI 的标准后台，我拿它来展示全栈交付、权限设计和部署能力。里面还带一个设备管理模块作为"二开"实例，用来证明这套模板能变成真正的业务后台。

> 🌐 英文版 / English version: [fastadmin-pro-admin.md](fastadmin-pro-admin.md)

---

## 背景

大多数小工厂和设备厂商都想要一个内部管理后台，而且要得快。市面上现成的模板基本是 Java 的（比如 RuoYi），要往定制化的 IoT 流程上改很别扭。我用 Python/FastAPI 自己做了一个，既贴合我实际的交付栈，也能当"后台二开"的复用底座。

---

## 问题

- 没有合适的 Python/FastAPI 后台起点拿给客户看。
- 很多模板的权限模型是写死的，不是数据驱动的。
- 审计日志要么没有，要么每个接口手写。
- 批量数据（设备、用户）没有干净的导入导出路子。

---

## 约束

| 约束 | 细节 |
|------|------|
| 技术栈契合 | Vue3 + FastAPI，贴合我的交付栈 |
| 权限模型 | 数据驱动 RBAC：目录/菜单/按钮，而不是散落在代码里判断 |
| 可审计 | 操作日志必须自动记录，不是每个接口各自写 |
| 演示安全 | 公网演示不能让访客把共享数据搞坏 |
| 可维护性 | 清晰分层：api / models / schemas / services / deps |

---

## 方案

```
┌──────────┐  /api   ┌────────────┐  SQLAlchemy  ┌─────────┐
│ Vue3 SPA │────────►│  FastAPI   │────────────►│  MySQL  │
│ Element  │ JWT Bearer│ RBAC      │             └─────────┘
│  Plus    │◄────────│ Excel/Log  │  Redis       ┌─────────┐
└──────────┘  JSON   └────────────┘────────────►│  Redis  │
                          │ Nginx 反代           └─────────┘
                          ▼
              docker-compose：mysql + redis + backend + frontend + nginx
```

整件事一条 `docker compose up -d` 就能起来——MySQL、Redis、FastAPI 后端、Nginx 托着的 Vue 构建、后边的 API，全在里面。

- **后端分层**：`api / core / deps / models / schemas / services` 加 `main.py`。业务逻辑放 `services`，路由保持薄。
- **鉴权**：PyJWT 的 access + refresh 双令牌；bcrypt 密码哈希；首次启动自动种一个 `admin` 超管。refresh token 放 Redis，能随时吊销。
- **RBAC**：`sys_user / sys_role / sys_menu`，菜单分 `catalog / menu / button` 三类。前端按菜单接口动态生成路由，按钮按权限码（如 `system:user:add`）显隐。
- **操作日志**：靠 FastAPI 依赖/中间件自动记——谁、方法、url、ip、模块、时间——业务接口不用手写日志。
- **软删除**：每张基础表都带 `deleted_at`；删除只是置位，查询默认过滤掉。
- **统一响应**：所有业务接口同一个信封：

```json
{ "code": 200, "message": "success", "data": { "list": [], "total": 0, "page": 1, "page_size": 10 } }
```

- **Excel**：用户和设备批量导入导出，逐行报错。
- **"二开"实例**：设备管理模块就是在同一套模板上长出来的——它展示从通用后台到具体业务后台（设备列表、状态、增删改查）的路子，而这正是大多数"后台二开"需求要干的事。

完整的设计规格（架构、库表、权限模型、接口契约）在项目的 `ForAI.md` 里。

---

## 关键技术决策

### 1. 双令牌，而不是单个 JWT
access token 短命，漏了损失也小；refresh token 在 Redis 里能吊销、能让前端静默续期。改动不大，但懂鉴权的人一看就知道是"认真做的"。

### 2. 菜单驱动前端，而不是反过来
把菜单（目录/菜单/按钮）存进表，非开发人员也能重新配权限，前端只从一个来源生成路由，不会有两套路由定义慢慢漂移。

### 3. 操作日志做成中间件，不写进每个接口
放进依赖，业务代码零污染，哪里都有一致的气迹。配好之后没人再操心它。

### 4. 默认软删除
每张基础表带 `deleted_at`，避免误删、也留着审计历史。查询顺手就过滤掉。

---

## 我的收获

一套模板能不能真的拿去二开，靠的是结构——deps → services → api、像样的权限模型、还有响应契约——不是某个单点功能。这几样对了，新的业务后台就是组装，不是考古。

---

## 标签

`#full-stack` `#fastapi` `#vue3` `#rbac` `#iot-admin`
