# ExpenseFlow — Architecture

A small PoC expense API. One journey: **submit** an expense in some currency →
**convert** it to the base currency (INR) on write → **approve or reject** it.

Stack: Python 3.12, FastAPI, Uvicorn, SQLAlchemy 2.0 on SQLite (`expenseflow.db`),
httpx for the external FX call, pydantic v2, pytest.

Two invariants drive the whole design:

- **Convert-then-insert atomically.** `amount_base_minor` is `NOT NULL` and is computed as
  part of the submit transaction. If FX is unavailable, the submit fails and no row is
  written — there is never a half-converted or null-base row.
- **Status is a terminal state machine** (`pending → approved | rejected`), enforced at the DB
  layer *and* via a conditional UPDATE, so an expense can be decided exactly once.

Money is always stored as integer minor units (paise/cents); conversion uses `Decimal`,
never float.

---

## 1) Schema — `expenses` table

| Column | Type | Constraints | Why it exists |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | Surrogate key; stable handle for `/expenses/{id}`. |
| `description` | TEXT | NOT NULL | What the expense is for. |
| `amount_original_minor` | INTEGER | NOT NULL, CHECK > 0 | Submitted amount in **minor units** of the original currency. Integer per the no-float-money rule; preserves exactly what was submitted. |
| `currency_original` | TEXT | NOT NULL | ISO-4217 3-letter code (e.g. `USD`). The source currency. |
| `amount_base_minor` | INTEGER | **NOT NULL** | Amount normalised to **INR minor units**, computed on write. `NOT NULL` guarantees no half-converted row can persist. |
| `fx_rate_micros` | INTEGER | NOT NULL | The exact rate used, stored as `rate × 1_000_000` (integer, no float). Makes each conversion reproducible/auditable. |
| `fx_rate_fetched_at` | TEXT (ISO-8601) | NOT NULL | When that rate was obtained. Pins the conversion to a moment in time. |
| `status` | TEXT | NOT NULL DEFAULT `'pending'`, CHECK IN (`pending`,`approved`,`rejected`) | The state machine; DB-level CHECK backstops the app logic. |
| `created_at` | TEXT (ISO-8601) | NOT NULL | Submit timestamp (audit / ordering). |
| `decided_at` | TEXT (ISO-8601) | NULL | Set once when approved/rejected. Null ⇔ still pending. |

**Money math:**
`amount_base_minor = round_half_up(amount_original_minor × fx_rate_micros / 1_000_000)`,
computed with `decimal.Decimal` and quantized to an integer — never float. Storing the
original amount, the rate, and the fetch time together means a base amount is fully
explainable after the fact and never silently changes if rates move.

---

## 2) Endpoints

All responses use the `ExpenseOut` shape unless noted.

| Method | Path | Request body | Success | Errors |
|---|---|---|---|---|
| `POST` | `/expenses` | `{ "description": str, "amount_minor": int>0, "currency": "USD" }` | **201** `ExpenseOut` (status=`pending`, base amount + rate filled in) | 422 bad input; **503** FX unavailable |
| `GET` | `/expenses` | — (optional `?status=pending\|approved\|rejected`) | **200** `[ExpenseOut]` | — |
| `GET` | `/expenses/{id}` | — | **200** `ExpenseOut` | 404 |
| `POST` | `/expenses/{id}/approve` | — | **200** `ExpenseOut` (status=`approved`, `decided_at` set) | 404 not found; **409** not `pending` |
| `POST` | `/expenses/{id}/reject` | — | **200** `ExpenseOut` (status=`rejected`, `decided_at` set) | 404; **409** not `pending` |

`ExpenseOut` = `{ id, description, amount_original_minor, currency_original,
amount_base_minor, fx_rate_micros, fx_rate_fetched_at, status, created_at, decided_at }`.

Conversion happens inside `POST /expenses` (normalised to base on write) — there is **no**
standalone convert endpoint. Approve/reject are separate action routes rather than a generic
`PATCH {status}` because the intent and the legal transitions are explicit.

---

## 3) File layout

- **`app/main.py`** — FastAPI app instance; `load_dotenv()`; lifespan running
  `Base.metadata.create_all`; includes the router; registers exception handlers mapping
  `FXUnavailable → 503` and `IllegalTransition → 409`.
- **`app/db.py`** — SQLAlchemy engine (`sqlite:///expenseflow.db`), `SessionLocal`, `Base`,
  and the `get_db` dependency.
- **`app/models.py`** — the `Expense` ORM model = the table in §1 (with `CHECK` constraints).
- **`app/schemas.py`** — pydantic v2 models: `ExpenseCreate` (request; validates
  `amount_minor > 0`, currency coerced to uppercase 3-letter) and `ExpenseOut` (response).
- **`app/routes.py`** — `APIRouter` with the five endpoints; orchestrates fx + db; enforces
  the state machine via a conditional UPDATE.
- **`app/fx.py`** — httpx client: `get_rate(currency) -> (rate_micros, fetched_at)`. Reads
  `FX_API_URL` / `FX_API_KEY` from env (no hardcoded secrets); applies a timeout and a
  bounded retry; raises `FXUnavailable` on failure. Isolated so it's mockable in tests and the
  failure path is contained.
- **`.env.example`** — documents `FX_API_URL`, `FX_API_KEY` (real `.env` untracked).
- **`tests/test_*.py`** — pytest; FX mocked so tests never hit the network.

---

## 4) Edge-case decisions

**A. FX API is down / slow at submit — what happens to `amount_base_minor`?**
Conversion is part of the write. `amount_base_minor` is `NOT NULL`; `app/fx.py` applies a
timeout + bounded retry and raises `FXUnavailable` on failure; the handler returns **503** and
the DB transaction never commits. The column is therefore **never written in a bad state** —
every stored row is fully converted and carries the exact rate + fetch time. Resubmitting
later is safe (no orphaned partial row).

**B. Can an already-approved expense be approved again?**
**No.** `status` is a terminal machine (`pending → approved | rejected`), enforced two ways: a
DB `CHECK` on the allowed values, and a **conditional UPDATE**
(`UPDATE expenses SET status=..., decided_at=... WHERE id=:id AND status='pending'`). The
handler inspects the affected-row count — 0 rows means either the id doesn't exist (**404**) or
it's already terminal (**409 Conflict**); >0 means the transition happened exactly once. This is
race-safe under concurrent approve calls without any app-level lock, and `decided_at` proves the
decision is one-time.

**C. Money rounding & currency minor-unit assumptions.**
Converting `minor × rate` yields fractional minor units; in float this drifts and breaks the
integer-money rule. We use `Decimal` with explicit round-half-up quantization to an integer, and
store `fx_rate_micros` so the result is reproducible. We do **not** assume every currency has 100
minor units — JPY has 0 decimals, KWD/BHD have 3 — so an implicit `×100` would be wrong. For the
PoC we validate `currency` against an allowlist of known-exponent codes (or store the exponent)
rather than hardcoding cents.

---

## Verification

- **Run:** `python -m uvicorn app.main:app --reload`
- **Happy path (FX mocked up):** `POST /expenses` → 201 with `amount_base_minor`,
  `fx_rate_micros`, `fx_rate_fetched_at` populated; `GET /expenses/{id}` reflects it.
- **Edge A:** point `FX_API_URL` at an unreachable host (or mock a timeout) → `POST /expenses`
  returns **503** and `GET /expenses` shows **no new row**.
- **Edge B:** approve a pending expense → 200; approve it again → **409**; reject an
  already-approved one → **409**; approve a non-existent id → **404**.
- **Edge C:** assert a known amount+rate rounds to the expected integer minor units (half-up),
  and that a `JPY`/`KWD`-style code is handled or rejected — not silently `×100`'d.
- **Automated:** `python -m pytest -q` with the FX client mocked.
