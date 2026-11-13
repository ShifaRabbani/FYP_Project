# Storefront: System Architecture

> **This is our rulebook.** Both team members (and Claude) follow it strictly.
> Any change to this file must be agreed by **both** members and noted in the
> [Change Log](#14-change-log) at the bottom.

**Version:** 1.0 · **Date:** 14 November 2026

---

## 1. The 6 Golden Rules

| # | Rule | Meaning in easy words |
|---|---|---|
| 1 | **The model proposes, the backend enforces** | The AI only *suggests* actions. The backend decides if they are allowed. |
| 2 | **The agent uses tools only** | The agent never touches the database directly. It can only call our tools. |
| 3 | **Tools call services only** | Tools contain no business logic. They call the same services the website uses. |
| 4 | **Sensitive = human approval** | Large refunds, cancellations, unclear customers → pause and ask a human. |
| 5 | **Everything is logged** | Every agent step, tool call, permission decision and approval goes to `audit_logs`. |
| 6 | **Core workflow first** | Order → stock → shipment → notify must work before RAG, styling or extras. |

---

## 2. High-Level Layers

```
┌───────────────────────────────────────────────┐
│  FRONTEND (React + Vite)                      │
│  Customer storefront  ·  Business dashboard   │
└──────────────────────┬────────────────────────┘
                       │ HTTPS + JWT
┌──────────────────────▼────────────────────────┐
│  API LAYER (FastAPI routers)                  │
│  Auth · Role check (RBAC) · Input validation  │
└──────────┬──────────────────────────┬─────────┘
           │                          │ /agent/run
           │                ┌─────────▼─────────┐     ┌──────────────┐
           │                │  AGENT (LangGraph)│◄────┤ RAG retriever│
           │                │  + LLM            │     │ (policies,   │
           │                └─────────┬─────────┘     │  FAQs)       │
           │                          │               └──────────────┘
           │                ┌─────────▼─────────┐
           │                │  TOOLS (6 typed)  │──► Approval gate
           │                └─────────┬─────────┘
┌──────────▼──────────────────────────▼─────────┐
│  SERVICES (all business rules live here)      │
└──────────────────────┬────────────────────────┘
┌──────────────────────▼────────────────────────┐
│  DATABASE (PostgreSQL)  + audit_logs          │
└───────────────────────────────────────────────┘
```

### Who may talk to whom (strict)

| Layer | ✅ May call | ❌ Must NOT call |
|---|---|---|
| Frontend | API only (through `src/api/`) | Database, LLM |
| Routers | Services, Agent | Database directly |
| Agent | Tools, RAG retriever | Services, Database |
| Tools | Services | Database directly, other tools |
| Services | Models / Database | Routers, Agent |
| RAG | Vector store | Business database |

---

## 3. Tech Stack

| Part | Choice | Note |
|---|---|---|
| Backend | FastAPI (Python 3.14) | Run with `python -m uvicorn` |
| Database | PostgreSQL | |
| ORM | SQLAlchemy 2.0 | Python classes ↔ tables |
| Migrations | Alembic | Never edit tables by hand |
| Validation | Pydantic | Request/response schemas |
| Auth | JWT + bcrypt password hashing | |
| Agent | LangGraph | Single agent first |
| LLM | *Decided in agent phase* | Keep it swappable (one config value) |
| RAG | Embeddings + vector store (*decided in RAG phase*) | |
| Frontend | React + Vite | |
| Routing | React Router | |
| HTTP | Axios | |
| Payments / courier | **Simulated** services | Real ones are future work |

---

## 4. Folder Structure (strict)

New files go **only** where this tree says.

```
FYP_Project/
├── backend/
│   ├── app/
│   │   ├── main.py              # creates FastAPI app, includes routers
│   │   ├── core/
│   │   │   ├── config.py        # reads .env settings
│   │   │   ├── database.py      # DB engine + session
│   │   │   ├── security.py      # JWT, password hashing
│   │   │   └── permissions.py   # role checks (RBAC)
│   │   ├── models/              # SQLAlchemy tables (1 file per table group)
│   │   ├── schemas/             # Pydantic request/response shapes
│   │   ├── routers/             # API endpoints only, no business logic
│   │   ├── services/            # ALL business logic + rules
│   │   ├── tools/               # 6 agent tools (thin wrappers over services)
│   │   ├── agent/
│   │   │   ├── state.py         # what the agent remembers during a run
│   │   │   ├── nodes.py         # interpret, check, execute, verify, report
│   │   │   ├── graph.py         # LangGraph wiring
│   │   │   └── prompts.py       # system prompt text
│   │   ├── rag/
│   │   │   ├── ingest.py        # chunk + embed documents
│   │   │   └── retriever.py     # find relevant chunks
│   │   └── baseline/
│   │       └── rule_based.py    # same workflow, fixed rules (for comparison)
│   ├── alembic/                 # database migrations
│   ├── tests/
│   ├── .env                     # secrets (NOT pushed)
│   ├── .env.example             # template of .env (pushed)
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── api/                 # all backend calls (client.js, products.js, ...)
│       ├── components/          # reusable UI pieces (Button, ProductCard, ...)
│       ├── pages/
│       │   ├── storefront/      # Home, ProductDetail, Cart, Checkout, OrderTracking, Login
│       │   └── dashboard/       # Overview, Products, Orders, AgentChat, Approvals, AuditLog
│       ├── context/             # AuthContext (logged-in user + role)
│       ├── routes/              # route list + role protection
│       ├── App.jsx
│       └── main.jsx
├── evaluation/
│   ├── test_cases.json          # fixed synthetic test set
│   └── results/                 # metric outputs
├── docs/                        # ERD, architecture, interviews, designs, report
├── .gitignore
└── README.md
```

> ⚠️ **Migration note:** today `main.py` is at `backend/main.py`. It moves to
> `backend/app/main.py` when we start the database step. The run command then becomes:
> `python -m uvicorn app.main:app --reload`

---

## 5. Roles & Permissions

| Action | Owner | Manager | Customer |
|---|---|---|---|
| Browse products, cart, checkout | ✅ | ✅ | ✅ |
| See own orders | ✅ | ✅ | ✅ |
| See all orders / customers | ✅ | ✅ | ❌ |
| Add / edit products & stock | ✅ | ✅ | ❌ |
| Use AI agent | ✅ | ✅ | ❌ |
| Approve sensitive actions | ✅ | ❌ | ❌ |
| View audit log | ✅ | ❌ | ❌ |

---

## 6. Agent Tools (the only things the agent can do)

| Tool | Input | Output | Allowed roles | Sensitive? |
|---|---|---|---|---|
| `search_product` | name / sku | product list | owner, manager | No |
| `check_inventory` | product_id, qty | available (yes/no), stock | owner, manager | No |
| `create_order` | customer_id, items | order_id | owner, manager | No |
| `calculate_order_total` | order_id | total | owner, manager | No |
| `create_shipment` | order_id | tracking_no (simulated) | owner, manager | No |
| `send_notification` | customer_id, order_id, message | sent (yes/no) | owner, manager | No |

**Sensitive actions** (always need owner approval): refunds above a set amount,
order cancellation, ambiguous customer match (2+ customers fit the request).

**Every tool must:** have typed inputs (Pydantic) · check the caller's role ·
call a service · return a clear result or error · write to `audit_logs`.

---

## 7. Agent Flow (LangGraph)

```
User goal
   ↓
[interpret]  → understand the goal, pick the next tool
   ↓
[check]      → is this role allowed? is it sensitive?
   ↓                    ↓ sensitive / unclear
[execute]            [ask_approval] → pause until owner approves/rejects
   ↓
[verify]     → did the tool really succeed? (re-read from DB)
   ↓
[decide]     → more steps? go back to interpret : go to report
   ↓
[report]     → tell the user what was done
```

| Situation | What the agent does |
|---|---|
| Out of stock | Stop, report which items + options |
| Tool fails | Retry **once**, then stop and tell the user |
| Sensitive or unclear | Pause for human approval |
| Not allowed for this role | Refuse and log as blocked |
| Max steps reached (10) | Stop and report |

---

## 8. Two Example Flows

**A) Customer places an order (no AI)**
`Checkout page → POST /orders → orders router → order_service (checks stock, saves order, reduces stock) → DB → response → Order tracking page`

**B) Manager uses the agent**
`AgentChat page → POST /agent/run → agent router → LangGraph → check_inventory → create_shipment → send_notification (each via services) → every step in audit_logs → final report shown in chat`

---

## 9. API Endpoints (v1)

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register` · `POST /auth/login` · `GET /auth/me` |
| Products | `GET /products` · `GET /products/{id}` · `POST /products` · `PUT /products/{id}` |
| Inventory | `GET /inventory` · `PUT /inventory/{product_id}` |
| Cart | `GET /cart` · `POST /cart/items` · `DELETE /cart/items/{id}` |
| Orders | `POST /orders` · `GET /orders` · `GET /orders/{id}` · `PUT /orders/{id}/status` |
| Shipments | `POST /shipments` · `GET /shipments/{order_id}` |
| Agent | `POST /agent/run` · `GET /agent/runs/{id}` |
| Approvals | `GET /approvals` · `POST /approvals/{id}/approve` · `POST /approvals/{id}/reject` |
| Audit | `GET /audit-logs` |
| RAG | `POST /assistant/ask` |

---

## 10. Database

The full ERD is in `docs/storefront_erd.dbml` (13 tables). Key rules:
- Every table has `id` (primary key) and `created_at`.
- Prices are `decimal`, never `float`.
- Status values are fixed lists (e.g. order: `pending, confirmed, shipped, delivered, cancelled`).
- Table changes only through **Alembic migrations**.

---

## 11. Coding Rules

| Topic | Rule |
|---|---|
| Python names | `snake_case` for files, functions, variables · `PascalCase` for classes |
| React names | `PascalCase.jsx` for components/pages · `camelCase` for functions |
| Secrets | Only in `.env`. Never in code, never pushed |
| Business logic | Only in `services/`. Never in routers, tools or React |
| Errors | Return clear messages (`"Product 12 is out of stock"`), not crashes |
| Comments | Short comment on top of every function: what it does |

---

## 12. Git Workflow

| Rule | Example |
|---|---|
| `main` must always run | Don't push broken code to `main` |
| One branch per feature | `feature/auth`, `feature/products-api`, `feature/agent-graph` |
| Commit message format | `type: short message` → `feat: add product list API`, `fix: stock check bug`, `docs: update README` |
| Pull before you start | `git pull` at the start of every session |
| Git commands | Run from `FYP_Project` (main folder) |
| Work log | Add today's work to `README.md` → Work Log at the end of each session |

---

## 13. Build Order

| Phase | What | Owner | Done when |
|---|---|---|---|
| 0 | Project setup | Both | ✅ Backend + React running (13–14 Nov) |
| 1 | DB connection + users table + auth (JWT) | A | Login works in `/docs` |
| 2 | Products, inventory, orders, shipments APIs | A | All endpoints tested in `/docs` |
| 3 | Storefront + dashboard pages | B | Customer can order from the website |
| 4 | Rule-based baseline script | A | Workflow runs with fixed rules |
| 5 | 6 tools + LangGraph agent + approval gate | A | Agent completes core workflow |
| 6 | Agent chat + approvals + audit pages | B | Manager can use agent from dashboard |
| 7 | RAG over policies/FAQs | B | Assistant answers from documents |
| 8 | Evaluation (6 metrics, 3 modes) + report | Both | Results table in `evaluation/results/` |

---

## 14. Change Log

| Date | Change | Agreed by |
|---|---|---|
| 14 Nov 2026 | Version 1.0 created | |
