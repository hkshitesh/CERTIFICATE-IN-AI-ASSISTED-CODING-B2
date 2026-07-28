# ExpenseFlow — How the Project Works (Plain-English Guide)

> This README is written for a **non-technical reader**. It explains, in everyday
> language, what each file in the project does and how they connect. No coding
> experience needed.

---

## 1. What is this project, in one sentence?

ExpenseFlow is a small piece of software (an "API") that lets someone **submit an
expense**, automatically **converts the money into Indian Rupees**, and then lets
an approver **approve or reject** it.

Think of it like the back-office engine behind a "Submit your travel bill" button.
It doesn't have buttons or screens itself — it's the machinery that a website or
app would talk to.

> **Important — current state of the project:**
> The project is **partly built**. The plan (see `docs/ARCHITECTURE.md`) describes
> six Python files, but **only three exist today**. This README describes what is
> **actually built right now**, and clearly marks what is still **planned but
> missing**. See the map in Section 3.

---

## 2. A quick mental model (the restaurant analogy)

Imagine a restaurant. The whole project is like the kitchen:

| Restaurant part | In this project | Built yet? |
|---|---|---|
| The **storage room / pantry** where ingredients are kept | The **database** (`expenseflow.db`) | ✅ Yes |
| The **rules for how the pantry is organised** (which shelf holds what) | `app/db.py` | ✅ Yes |
| The **recipe cards** describing exactly what an "expense" looks like | `app/models.py` | ✅ Yes |
| The **front door and waiter** who takes your order | `app/main.py` | ❌ Not built yet |
| The **order-checking clerk** who makes sure your order form is filled correctly | `app/schemas.py` | ❌ Not built yet |
| The **chefs** who actually do the work (convert currency, approve, reject) | `app/routes.py` | ❌ Not built yet |

Right now, the **pantry and its organising rules exist**, but the **waiters and
chefs have not been hired yet**. So the project can *store* expenses correctly,
but it cannot yet *receive* them from the outside world.

---

## 3. The file map (what exists today)

```
expenseflow/
│
├── app/                      ← the actual program lives here
│   ├── __init__.py           ✅ EXISTS  — marks "app" as one program package
│   ├── db.py                 ✅ EXISTS  — sets up and connects to the database
│   ├── models.py             ✅ EXISTS  — defines what an "expense" looks like
│   │
│   ├── main.py               ❌ PLANNED — the front door / startup file
│   ├── schemas.py            ❌ PLANNED — checks incoming request forms
│   └── routes.py             ❌ PLANNED — the actual expense actions
│
├── expenseflow.db            ✅ EXISTS  — the database file (the "pantry")
├── docs/ARCHITECTURE.md      ✅ EXISTS  — the detailed design blueprint
├── CLAUDE.md                 ✅ EXISTS  — house rules for how the code is written
└── README.md                 ← this file
```

---

## 4. What each existing file does (in plain English)

###  `app/__init__.py` — "This folder is one program"

This is a nearly empty file. Its only job is to tell Python:
*"Treat everything in the `app` folder as one connected program, not a random pile
of files."*

Because this file exists, the other files are allowed to say things like
*"borrow the database setup from `app.db`"*. Without it, the files couldn't find
each other.

**Think of it as:** the label on the outside of a box that says "Kitchen — these
things belong together."

---

###  `app/db.py` — "The connection to the storage room"

This file is the **link between the program and the database file**
(`expenseflow.db`). It does four main things:

1. **Finds the database.** It looks up *where* the database lives from a setting
   called `DATABASE_URL`. If nobody specifies one, it defaults to the local file
   `expenseflow.db`. (This "read the location from a setting" approach means no
   sensitive details are hard-baked into the code.)

2. **Opens a connection** to that database (the "engine").

3. **Hands out short-term work sessions.** Every time someone submits or reads an
   expense, this file gives out a temporary "session" — like giving a clerk a
   clipboard to do one task — and then **always cleans it up afterwards**, even if
   something goes wrong.

4. **Can build the tables.** It has a routine (`init_db`) that creates the storage
   shelves (the tables) inside the database if they don't already exist. It's safe
   to run repeatedly — it won't wipe existing data.

**Think of it as:** the pantry manager who knows where the storage room is, opens
the door, hands staff a basket to fetch things, and locks up after them.

---

###  `app/models.py` — "The definition of an expense"

This file describes, in exact detail, **what one expense record looks like** and
what rules it must obey. It depends on `db.py` (it borrows the "Base" foundation
from there), so `db.py` must exist first.

It defines two things:

**a) The list of allowed statuses** — an expense can only ever be one of:
- `pending` — submitted, waiting for a decision
- `approved` — accepted (final, can't change)
- `rejected` — declined (final, can't change)

**b) The "expense" record itself** — every expense stores these details:

| Stored detail | Plain meaning |
|---|---|
| `id` | A unique ticket number for each expense |
| `amount_original_minor` | The original amount submitted (e.g. in US dollars) |
| `currency_original` | Which currency it was submitted in (e.g. "USD") |
| `amount_base_minor` | The same money **converted into Indian Rupees** |
| `fx_rate_ppm` | The exact exchange rate that was used (kept for the record) |
| `fx_rate_at` | The date/time that exchange rate was fetched |
| `status` | pending / approved / rejected |
| `description` | Optional note (e.g. "Client dinner") |
| `category` | Optional tag (e.g. "travel", "meals") |
| `reject_reason` | Why it was rejected (only filled in if rejected) |
| `created_at` | When it was submitted |
| `decided_at` | When it was approved or rejected |

This file also sets up **safety rules** that the database itself enforces — for
example, amounts must be greater than zero, and the status must be one of the three
allowed words. Even if some future code makes a mistake, the database will refuse to
store bad data.

> **Two deliberate design choices worth knowing:**
> - **Money is stored as whole numbers, never decimals.** ₹10.50 is stored as
>   `1050` ("paise", the smallest coin unit). This avoids tiny rounding errors that
>   decimals can cause with money.
> - **Every expense must have a rupee value.** An expense cannot be saved unless the
>   currency conversion succeeded. If the exchange-rate service is down, nothing is
>   saved at all (rather than saving something incomplete).

**Think of it as:** the official blank form + the rulebook that says which boxes are
required and what counts as valid.

---

## 5. How the existing files work together

Here is the chain of dependencies among the **three files that exist today**.
An arrow means *"needs / borrows from"*:

```
   app/models.py   ──needs──▶   app/db.py   ──talks to──▶   expenseflow.db
  (the expense form)          (the DB link)              (the storage file)

   app/__init__.py  ──── ties the "app" folder together so the above can find each other
```

Read it left to right:

1. **`db.py`** knows how to reach the storage file `expenseflow.db`.
2. **`models.py`** borrows the foundation from `db.py` and uses it to describe the
   "expense" record.
3. **`__init__.py`** quietly makes sure all these files count as one program so they
   can reference each other.

So today the project is a **well-defined, well-guarded filing system**: it knows
exactly what an expense is and how to store it safely. What it's **missing** is the
part that lets the outside world actually send in expenses and take actions.

---

## 6. What's still missing (the planned files)

According to `docs/ARCHITECTURE.md`, three more files are meant to be added to make
the project fully usable:

| Missing file | What it will add |
|---|---|
| `app/main.py` | The **starting point / front door** — turns the code into a running service that a website or app can connect to. |
| `app/schemas.py` | The **form-checker** — verifies incoming requests are filled out correctly (valid amount, valid 3-letter currency) before anything is stored. |
| `app/routes.py` | The **action handlers** — the actual logic for *submit an expense*, *convert its currency*, *approve*, *reject*, and *view* expenses. This is where the currency conversion happens. |

Once these exist, the full journey will work end to end:

```
Someone submits an expense
        │
        ▼
 schemas.py checks the form is valid
        │
        ▼
 routes.py fetches today's exchange rate and converts the money to INR
        │
        ▼
 models.py + db.py store the finished expense in expenseflow.db
        │
        ▼
 Later: an approver calls "approve" or "reject" → routes.py updates the record
```

---

## 7. The intended end-to-end journey (once complete)

When all files are built, the full story will be:

1. **Submit** — A user sends in an expense (amount + currency + optional notes).
2. **Validate** — The form is checked for obvious mistakes.
3. **Convert** — The system looks up the live exchange rate and converts the amount
   into Indian Rupees. *If the exchange-rate service is unavailable, the submission
   is rejected and nothing is saved* — no half-finished records.
4. **Store** — The expense is saved as `pending`.
5. **Decide** — An approver either **approves** or **rejects** it. This can happen
   only once; a decided expense can't be changed again.

---

## 8. Where to look for more detail

- **`docs/ARCHITECTURE.md`** — the full technical blueprint, including the exact
  database columns, the planned web addresses (endpoints), and the reasoning behind
  the tricky decisions (like what happens when the exchange-rate service fails).
- **`CLAUDE.md`** — the "house rules" for how this code must be written (e.g. money
  as whole numbers, no secrets in the code, base currency is INR).

---

*This README describes the project as of the current state: `__init__.py`, `db.py`,
and `models.py` are built; `main.py`, `schemas.py`, and `routes.py` are planned but
not yet written.*
