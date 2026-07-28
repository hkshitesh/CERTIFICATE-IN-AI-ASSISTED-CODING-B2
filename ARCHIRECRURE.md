# ExpenseFlow — Architecture

## Overview
ExpenseFlow is a PoC expense submission-and-approval API (FastAPI + SQLAlchemy on
SQLite). One journey: **submit** an expense in some currency → it is **converted** to
the base currency (**INR**) on write via an external FX call → an approver **approves or
rejects** it.

Design invariants:
- Money is stored only as **integer minor units** (paise/cents) — never float.
- Base currency is **INR**; every stored expense carries a valid INR amount.
- Secrets and config come from environment variables (python-dotenv).

---

## 1. Database schema — `expenses` table

| Column | Type | Constraints | Why it exists |
|---|---|---|---|
| `id` | INTEGER | PK, AUTOINCREMENT | Surrogate key; stable handle for `/expenses/{id}` routes. |
| `amount_original_minor` | INTEGER | NOT NULL, `CHECK > 0` | The submitted amount in the **original** currency's minor units. Integer, never float. |
| `currency_original` | TEXT | NOT NULL, `CHECK length = 3` | ISO-4217 code of what was submitted (e.g. `USD`). The original is never lost after normalisation. |
| `amount_base_minor` | INTEGER | **NOT NULL**, `CHECK > 0` | The normalised **INR paise** value. NOT NULL is the core invariant — a row can only exist with a real base amount (see §4.A). |
| `fx_rate_ppm` | INTEGER | NOT NULL | The exact rate applied, stored as **parts-per-million** (`rate × 1_000_000`) so it stays integer — no float in the DB. Audit trail of what rate was actually used. |
| `fx_rate_at` | TEXT (ISO-8601) | NOT NULL | When that rate was fetched. Staleness / audit. |
| `status` | TEXT | NOT NULL, DEFAULT `'PENDING'`, `CHECK IN ('PENDING','APPROVED','REJECTED')` | The state machine (see §4.B). |
| `description` | TEXT | NULL | Optional free-text memo. |
| `category` | TEXT | NULL | Optional tag (travel, meals, …). |
| `reject_reason` | TEXT | NULL | Populated only on reject; audit of why. |
| `created_at` | TEXT (ISO-8601) | NOT NULL, DEFAULT now | Submission time. Audit / ordering. |
| `decided_at` | TEXT (ISO-8601) | NULL | Set exactly once, when approved or rejected. NULL while PENDING — also marks "has been decided". |

Notes:
- `fx_rate_ppm` + `amount_original_minor` + `amount_base_minor` together let the
  conversion be re-derived and verified later without another API call.
- Money `> 0` checks and the `status` `CHECK` are enforced at the DB layer so bad data
  can't exist even if application logic regresses.

---

## 2. Endpoints

Conversion to INR happens **on submit** ("normalised to base on write"). Approve and
reject are two explicit routes. Read endpoints exist so the flow is verifiable.

| Method | Path | Request body | Success | Errors |
|---|---|---|---|---|
| POST | `/expenses` | `{ amount_minor, currency, description?, category? }` | **201** `ExpenseOut` (PENDING) | **422** invalid body · **503** FX API down (nothing stored) |
| GET | `/expenses` | — (`?status=` optional) | **200** `[ExpenseOut]`, newest first | — |
| GET | `/expenses/{id}` | — | **200** `ExpenseOut` | **404** not found |
| POST | `/expenses/{id}/approve` | none | **200** `ExpenseOut` (APPROVED) | **404** unknown id · **409** already decided |
| POST | `/expenses/{id}/reject` | `{ reason? }` | **200** `ExpenseOut` (REJECTED) | **404** unknown id · **409** already decided |

`ExpenseOut` shape: `id, amount_original_minor, currency_original, amount_base_minor,
base_currency ("INR"), fx_rate_ppm, fx_rate_at, status, description, category,
reject_reason, created_at, decided_at`.

---

## 3. File layout

| File | Responsibility |
|---|---|
| `app/main.py` | Creates the FastAPI app, loads `.env`, includes the router, creates tables on startup (`Base.metadata.create_all`). Entry for `uvicorn app.main:app`. |
| `app/db.py` | SQLAlchemy `engine` (SQLite, `DATABASE_URL` from env), `SessionLocal`, declarative `Base`, and a `get_db()` dependency yielding a scoped session. |
| `app/models.py` | The `Expense` ORM model + `Status` enum + `CHECK` constraints. |
| `app/schemas.py` | pydantic v2 models: `ExpenseCreate`, `ExpenseOut`, `RejectRequest`, `StatusEnum`. Input validation (positive amount, 3-letter currency). |
| `app/routes.py` | The `APIRouter` with the five endpoints (docstring each), plus the private FX-fetch helper `_fetch_rate_ppm(currency)` and the integer conversion function. |
| `tests/test_expenses.py` | pytest covering the journey and both edge cases; FX call mocked — no live network. |
| `.env.example` (committed) / `.env` (git-ignored) | `BASE_CURRENCY=INR`, `FX_API_URL`, `FX_API_KEY`, `DATABASE_URL=sqlite:///./expenseflow.db`, `FX_TIMEOUT_SECONDS`. |
| `requirements.txt` | Pins the deps already in `.venv` (adds no new library). |

FX logic lives inside `routes.py` to keep the CLAUDE.md 5-file layout; it can be
extracted to `app/fx.py` later if it grows.

---

## 4. Edge-case decisions

### A. FX API is down when an expense is submitted — **fail closed**
`amount_base_minor` is `NOT NULL`, and the row is inserted **only after** the rate is in
hand, inside a single transaction. The submit handler fetches the rate under a bounded
timeout (`FX_TIMEOUT_SECONDS`, e.g. 5s) with one short retry; on timeout, connection
error, or non-200 it returns **503 and persists nothing**.

- `amount_base_minor` is never written NULL, zero, or as a float placeholder — there is
  no partial row and no reconciliation job. The caller simply retries later.
- **Why:** the invariant *"every stored expense has a real INR amount"* keeps approval
  logic trivial (an approver always sees a true figure) and avoids a `PENDING_CONVERSION`
  substate + backfill worker — overkill for a PoC.
- *Alternative considered and rejected:* nullable base amount + `PENDING_CONVERSION`
  status + retry worker; more resilient, more moving parts.

### B. Approving an already-decided expense — **terminal state machine, 409 on repeat**
```
PENDING --approve--> APPROVED (terminal)
PENDING --reject --> REJECTED (terminal)
APPROVED / REJECTED --any action--> 409 Conflict
```
- APPROVED and REJECTED are terminal. A repeat approve — or approve-then-reject — returns
  **409 Conflict** and changes nothing. `status`/`decided_at` are written exactly once.
- **Race-safe** via an atomic conditional update rather than read-then-write:
  `UPDATE expenses SET status='APPROVED', decided_at=:now WHERE id=:id AND status='PENDING'`.
  - `rowcount == 1` → **200** (won the transition).
  - `rowcount == 0` → follow-up `SELECT`: absent → **404**, present-but-decided → **409**.
- *Alternative considered and rejected:* idempotent re-approve (APPROVED + approve → 200
  no-op); a distinct 409 gives approvers a clearer audit signal that the action already happened.

### C. Currency minor-unit exponent mismatch
Conversion is integer-only:
`amount_base_minor = round(amount_original_minor × fx_rate_ppm / 1_000_000)` with a single
documented rounding rule — never Python `float` on stored values.
- **The trap:** minor-unit exponents differ (USD/EUR = 2 decimals, **JPY = 0**, KWD = 3).
  Assuming a universal "cents" exponent silently mis-scales JPY/KWD by 100×.
- **PoC handling:** validate `currency` against an allowlist of 2-decimal currencies and
  reject others with **422**, so a mis-scaled base amount is never stored. A full
  currency-exponent table is the production follow-up.

---

## Run & test
- Run: `python -m uvicorn app.main:app --reload`
- Test: `python -m pytest -q` — covers submit→approve, submit→reject, FX-down → 503 (empty
  DB), double-approve → 409, approve-then-reject → 409, unknown id → 404, and an integer
  rounding assertion. FX is mocked; no live network.
