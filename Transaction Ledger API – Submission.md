# Transaction Ledger API

Python · Flask · SQLite — 36 passing tests

**Run locally:** `pip install -r requirements.txt` then `python run.py` (server on `http://127.0.0.1:5000`). Run tests with `python -m unittest discover -v`.\
Copy the files below into the structure shown in the README (`app/`, `tests/`, `docs/`). Tap a file name to expand it.

README.md

````
# Transaction Ledger API

A small backend for a transaction ledger: create accounts, post credit/debit
transactions, read balances and paginated history, and (bonus) transfer money
between accounts atomically.

**Stack:** Python 3.10+, Flask, SQLite (standard library `sqlite3`).
No ORM and no other dependencies, so it runs anywhere with one `pip install`.

## Run locally

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python run.py                      # http://127.0.0.1:5000
```

The SQLite file (`ledger.db`) and all tables are created automatically on first start.

| Env var    | Default      | Purpose                    |
|------------|--------------|----------------------------|
| `DATABASE` | `ledger.db`  | Path of the SQLite file    |
| `HOST`     | `127.0.0.1`  | Bind address               |
| `PORT`     | `5000`       | Port                       |

### Quick try

```bash
curl -s -XPOST localhost:5000/accounts -H 'Content-Type: application/json' -d '{"name":"Alice"}'
# use the returned id:
curl -s -XPOST localhost:5000/accounts/<ID>/transactions -H 'Content-Type: application/json' \
  -d '{"type":"credit","amount":100,"reference_id":"dep-1","description":"Deposit"}'
curl -s "localhost:5000/accounts/<ID>/transactions?page=1&limit=10"
```

### Run the tests

```bash
python -m unittest discover -v
```

36 tests cover validation, balance rules, duplicate handling, pagination/order,
immutability (API and DB level), atomic rollback (with injected failures),
concurrent debits, and transfers. (Stack traces in the output come from the
deliberately injected failures in the atomicity tests.)

## Documentation

- [`docs/API.md`](docs/API.md) – endpoints, request/response examples, error catalogue
- [`ASSUMPTIONS.md`](ASSUMPTIONS.md) – interpretation choices and trade-offs

## Project structure

```
app/
  __init__.py     app factory (config, DB init, error handlers, blueprint)
  routes.py       HTTP layer only: parse -> validate -> call service -> JSON
  validation.py   request validation, collects all field errors at once
  services.py     business logic; every money movement happens here
  db.py           connections, transaction context managers (BEGIN IMMEDIATE)
  money.py        decimal <-> integer-cents conversion
  errors.py       exception types + one consistent error JSON shape
  schema.sql      tables, constraints, immutability triggers
tests/test_api.py
run.py
```

## How data integrity is guaranteed

| Requirement | Mechanism |
|---|---|
| **Atomicity** | Inserting the transaction row and updating the balance run inside one `BEGIN IMMEDIATE … COMMIT`. Any exception triggers `ROLLBACK`, so neither change persists. Transfers put both legs (and the transfer record) in a single DB transaction. |
| **Balance consistency** | Balance changes only inside the same DB transaction that inserts the ledger row, with the debit funds-check performed under the write lock. A failed/rejected request rolls back, so the balance always equals Σcredits − Σdebits (asserted in tests). |
| **No overdraft / race safety** | `BEGIN IMMEDIATE` serialises writers; `CHECK (balance_minor >= 0)` is a last line of defence. A test fires 25 concurrent debits of 10 against a balance of 100: exactly 10 succeed. |
| **Immutability** | No PUT/PATCH/DELETE routes exist (→ `405`). DB triggers also `RAISE(ABORT)` on any `UPDATE`/`DELETE` of `transactions` and `transfers`, so even direct SQL cannot change history. |
| **Duplicate reference** | `UNIQUE (account_id, reference_id)` enforced by the database, surfaced as `409 DUPLICATE_TRANSACTION`. |
| **Money precision** | Stored as integer cents; amounts accepted with ≤ 2 decimals; returned as strings (`"100.00"`). No floats in arithmetic. |
| **Initial balance** | The insert hard-codes `0`; a client-supplied `balance` is rejected with a validation error. |

## Design notes

- **Why SQLite?** Zero setup for reviewers, real ACID transactions, CHECK/UNIQUE/FK constraints and triggers. The service layer is plain SQL; porting to PostgreSQL means swapping `BEGIN IMMEDIATE` for `SELECT … FOR UPDATE` on the account row(s), locking in a consistent id order to avoid deadlocks in transfers.
- **Thin routes, fat services:** routes hold no business logic, so the rules are testable in one place.
- **Errors:** every error, including 404/405/500, uses `{"error": {"code", "message", "details"}}`.
````

docs/API.md

````
# API Documentation

Base URL: `http://127.0.0.1:5000` · All bodies are JSON (`Content-Type: application/json`).
Amounts are sent as a number or numeric string (max 2 decimals) and **returned as strings**.

| Method | Path | Description |
|---|---|---|
| POST | `/accounts` | Create an account |
| GET | `/accounts` | List accounts (paginated, newest first) |
| GET | `/accounts/{account_id}` | Get account and current balance |
| POST | `/accounts/{account_id}/transactions` | Add a credit/debit transaction |
| GET | `/accounts/{account_id}/transactions` | Transaction history (paginated, latest first) |
| GET | `/accounts/{account_id}/transactions/{transaction_id}` | Get one transaction |
| POST | `/transfers` | Transfer between two accounts (atomic, bonus) |
| GET | `/health` | Liveness check |

There are intentionally **no** update/delete endpoints for transactions (`PUT/PATCH/DELETE` → `405`).

---

## Create account — `POST /accounts`

```json
{ "name": "Alice" }
```
| Field | Rules |
|---|---|
| `name` | required, string, non-blank, ≤ 100 chars |

Any other field (`balance`, `id`, `created_at`, …) is rejected with `422`.

`201 Created` (+ `Location` header)
```json
{ "id": "83de428f-08aa-46d8-b689-b486d1a4fa92", "name": "Alice",
  "balance": "0.00", "created_at": "2026-10-26T13:44:10.326672Z" }
```

## Get account — `GET /accounts/{account_id}`

`200 OK` with the same shape. Errors: `400 INVALID_ACCOUNT_ID`, `404 ACCOUNT_NOT_FOUND`.

## Add transaction — `POST /accounts/{account_id}/transactions`

```json
{ "type": "credit", "amount": 50.25, "reference_id": "dep-001", "description": "Salary" }
```
| Field | Rules |
|---|---|
| `type` | required, exactly `"credit"` or `"debit"` |
| `amount` | required, > 0, ≤ 2 decimals, ≤ 1,000,000,000 |
| `reference_id` | required, non-blank string ≤ 100 chars, unique per account |
| `description` | optional string ≤ 500 chars |

`201 Created`
```json
{
  "transaction": {
    "id": "e3a1a070-c215-47ed-83ac-494c878b723a",
    "account_id": "83de428f-08aa-46d8-b689-b486d1a4fa92",
    "type": "credit", "amount": "50.25", "description": "Salary",
    "reference_id": "dep-001", "transfer_id": null,
    "created_at": "2026-10-26T13:44:10.377753Z"
  },
  "account": { "id": "83de428f-…", "name": "Alice", "balance": "50.25",
               "created_at": "2026-10-26T13:44:10.326672Z" }
}
```
Errors: `400 INVALID_ACCOUNT_ID` · `400 INVALID_REQUEST_BODY` · `404 ACCOUNT_NOT_FOUND` · `409 DUPLICATE_TRANSACTION` · `422 VALIDATION_ERROR` · `422 INSUFFICIENT_BALANCE`

## List transactions — `GET /accounts/{account_id}/transactions?page=1&limit=20`

| Param | Default | Rules |
|---|---|---|
| `page` | 1 | integer ≥ 1 |
| `limit` | 20 | integer 1–100 |

Sorted by `created_at` **descending** (latest first).

`200 OK`
```json
{
  "data": [ { "id": "…", "type": "debit", "amount": "20.00", "reference_id": "…", "…": "…" } ],
  "pagination": { "page": 1, "limit": 20, "total_records": 4, "total_pages": 1 }
}
```
Errors: `400 INVALID_PAGINATION`, `400 INVALID_ACCOUNT_ID`, `404 ACCOUNT_NOT_FOUND`.

## Transfer — `POST /transfers` (bonus)

```json
{ "from_account_id": "<uuid>", "to_account_id": "<uuid>", "amount": 30,
  "reference_id": "xfer-001", "description": "Rent share" }
```
`reference_id` is required and unique per source account. `description` is optional.

`201 Created`
```json
{
  "transfer": { "id": "…", "from_account_id": "…", "to_account_id": "…", "amount": "30.00",
                "description": "Rent share", "reference_id": "xfer-001", "created_at": "…" },
  "from_account": { "id": "…", "balance": "70.00", "…": "…" },
  "to_account":   { "id": "…", "balance": "80.00", "…": "…" }
}
```
The debit, the credit and the transfer record commit together or not at all.
Errors: `404 ACCOUNT_NOT_FOUND` (details name the offending field) · `409 DUPLICATE_TRANSACTION` · `422 VALIDATION_ERROR` (including same account) · `422 INSUFFICIENT_BALANCE`.

---

## Error format

Every error response has the same shape:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "details": [
      { "field": "amount", "code": "INVALID_AMOUNT", "message": "amount must be greater than 0" },
      { "field": "type", "code": "INVALID_TRANSACTION_TYPE", "message": "type must be either 'credit' or 'debit'." }
    ]
  }
}
```
Validation reports **all** invalid fields at once.

| HTTP | `error.code` | When |
|---|---|---|
| 400 | `INVALID_REQUEST_BODY` | Body missing, not valid JSON, or not a JSON object |
| 400 | `INVALID_ACCOUNT_ID` | Account id is not a valid UUID |
| 400 | `INVALID_PAGINATION` | `page`/`limit` not a positive integer, or `limit` > 100 |
| 404 | `ACCOUNT_NOT_FOUND` | Well-formed id, no such account |
| 404 | `TRANSACTION_NOT_FOUND` | No such transaction on this account |
| 404 | `NOT_FOUND` | Unknown route |
| 405 | `METHOD_NOT_ALLOWED` | e.g. editing/deleting a transaction |
| 409 | `DUPLICATE_TRANSACTION` | `reference_id` already used on this account (or source account, for transfers) |
| 422 | `VALIDATION_ERROR` | Field-level problems; see `details[].code`: `REQUIRED`, `INVALID_TYPE`, `TOO_LONG`, `UNKNOWN_FIELD`, `INVALID_TRANSACTION_TYPE`, `INVALID_AMOUNT`, `INVALID_ACCOUNT_ID`, `SAME_ACCOUNT` |
| 422 | `INSUFFICIENT_BALANCE` | Debit/transfer exceeds available balance |
| 500 | `INTERNAL_ERROR` | Unexpected failure; any partial work is rolled back |
````

ASSUMPTIONS.md

```
# Assumptions & Decisions

1. **Single currency, 2 decimal places.** No currency field. Amounts have at most 2 decimals and are capped at 1,000,000,000.00 per transaction.
2. **Amounts in JSON** may be a number or a numeric string; they are always **returned as strings** (`"100.00"`) so clients never lose precision.
3. **Balances cannot go negative.** A debit is allowed only when `balance >= amount` (spending the exact balance is fine).
4. **IDs are UUID v4**, generated server-side. A malformed account id → `400 INVALID_ACCOUNT_ID`; a well-formed but unknown id → `404 ACCOUNT_NOT_FOUND`.
5. **Timestamps** are UTC, ISO-8601 with microseconds and a `Z` suffix, generated by the server. History is ordered by `created_at DESC` with insertion order as a tie-breaker.
6. **`reference_id`** is a required, trimmed, non-empty string (≤ 100 chars), unique **per account**, shared across credit and debit. The same value may be used on different accounts. Matching is case-sensitive.
7. **Duplicates are rejected, not replayed.** Re-sending an existing `reference_id` returns `409 DUPLICATE_TRANSACTION` and never changes the balance. (An alternative design returns the original transaction for identical retries; I chose the stricter, simpler behaviour.) A request that failed (validation, insufficient funds, error) does not consume its `reference_id`.
8. **Strict input.** Unknown fields are rejected with 422, which is how "client cannot set the initial balance" is enforced explicitly rather than silently ignored. Strings are trimmed; `type` is case-sensitive (`credit` / `debit`).
9. **Status codes:** `400` malformed JSON/body, bad ids, bad pagination; `422` semantic validation and insufficient balance; `404` not found; `409` duplicate; `405` unsupported methods (e.g. editing transactions); `500` unexpected (no internals leaked).
10. **Pagination:** `page` (default 1) and `limit` (default 20, max 100), both positive integers. A page beyond the last returns an empty `data` array with correct metadata (not an error). `total_pages` is 0 when there are no records.
11. **Accounts are never deleted or renamed** (no endpoint required), so accounts and transactions are effectively append-only.
12. **Transfers**
    - Require a client `reference_id`, unique per source account (duplicate protection).
    - Create two ordinary ledger entries (debit on source, credit on destination) linked by `transfer_id`, with system-generated references `transfer:<transfer_id>:debit|credit`, so transfers appear in each account's history and the balance invariant still holds.
    - Source and destination must differ.
13. **No authentication/authorisation**: out of scope for the assignment. In production, accounts would belong to an authenticated owner.
14. **SQLite concurrency:** SQLite permits one writer at a time, which is what gives race-free balance checks here. For high write volume, use PostgreSQL with row locks.
```

app/\_\_init\_\_.py

```
import os

from flask import Flask

from . import db
from .errors import register_error_handlers
from .routes import bp


def create_app(config=None):
    app = Flask(__name__)
    app.config["DATABASE"] = os.environ.get("DATABASE", "ledger.db")
    app.json.sort_keys = False
    if config:
        app.config.update(config)

    db.init_db(app.config["DATABASE"])
    app.teardown_appcontext(db.close_db)
    register_error_handlers(app)
    app.register_blueprint(bp)
    return app
```

app/routes.py

```
from flask import Blueprint, jsonify, request

from . import services, validation
from .db import get_db
from .errors import InvalidRequestBody

bp = Blueprint("api", __name__)


def _json_body():
    body = request.get_json(force=True, silent=True)
    if body is None:
        raise InvalidRequestBody("Request body must be valid JSON.")
    return body


@bp.get("/health")
def health():
    return jsonify({"status": "ok"})


@bp.post("/accounts")
def create_account():
    data = validation.validate_create_account(_json_body())
    account = services.create_account(get_db(), data["name"])
    return jsonify(account), 201, {"Location": f"/accounts/{account['id']}"}


@bp.get("/accounts")
def list_accounts():
    page, limit = validation.parse_pagination(request.args)
    return jsonify(services.list_accounts(get_db(), page, limit))


@bp.get("/accounts/<account_id>")
def get_account(account_id):
    account_id = validation.validate_uuid(account_id)
    return jsonify(services.get_account(get_db(), account_id))


@bp.post("/accounts/<account_id>/transactions")
def add_transaction(account_id):
    account_id = validation.validate_uuid(account_id)
    data = validation.validate_create_transaction(_json_body())
    result = services.add_transaction(get_db(), account_id, data["type"], data["amount_minor"],
                                      data["description"], data["reference_id"])
    return jsonify(result), 201


@bp.get("/accounts/<account_id>/transactions")
def list_transactions(account_id):
    account_id = validation.validate_uuid(account_id)
    page, limit = validation.parse_pagination(request.args)
    return jsonify(services.list_transactions(get_db(), account_id, page, limit))


@bp.get("/accounts/<account_id>/transactions/<tx_id>")
def get_transaction(account_id, tx_id):
    account_id = validation.validate_uuid(account_id)
    return jsonify(services.get_transaction(get_db(), account_id, tx_id))


@bp.post("/transfers")
def create_transfer():
    d = validation.validate_create_transfer(_json_body())
    result = services.create_transfer(get_db(), d["from_account_id"], d["to_account_id"],
                                      d["amount_minor"], d["description"], d["reference_id"])
    return jsonify(result), 201
```

app/validation.py

```
"""Request validation. Each validator returns clean values or raises ValidationError."""
import uuid

from .errors import InvalidAccountId, InvalidPagination, InvalidRequestBody, ValidationError
from .money import parse_amount

TRANSACTION_TYPES = ("credit", "debit")
MAX_NAME = 100
MAX_DESCRIPTION = 500
MAX_REFERENCE = 100
DEFAULT_LIMIT, MAX_LIMIT = 20, 100


def _require_object(body):
    if not isinstance(body, dict):
        raise InvalidRequestBody("Request body must be a JSON object.")


def _reject_unknown(body, allowed, errors):
    for key in body:
        if key not in allowed:
            errors.append({"field": key, "code": "UNKNOWN_FIELD",
                           "message": f"'{key}' is not allowed or is set by the system."})


def _string(body, field, errors, *, required, max_len):
    value = body.get(field)
    if value is None:
        if required:
            errors.append({"field": field, "code": "REQUIRED", "message": f"{field} is required."})
        return None
    if not isinstance(value, str):
        errors.append({"field": field, "code": "INVALID_TYPE", "message": f"{field} must be a string."})
        return None
    value = value.strip()
    if required and not value:
        errors.append({"field": field, "code": "REQUIRED", "message": f"{field} must not be empty."})
        return None
    if len(value) > max_len:
        errors.append({"field": field, "code": "TOO_LONG",
                       "message": f"{field} must be at most {max_len} characters."})
        return None
    return value


def _amount(body, errors, field="amount"):
    if body.get(field) is None:
        errors.append({"field": field, "code": "REQUIRED", "message": f"{field} is required."})
        return None
    try:
        return parse_amount(body[field])
    except ValueError as exc:
        errors.append({"field": field, "code": "INVALID_AMOUNT", "message": str(exc)})
        return None


def _finish(errors):
    if errors:
        raise ValidationError("Request validation failed.", errors)


def validate_uuid(value, field="account_id"):
    try:
        return str(uuid.UUID(str(value)))
    except ValueError:
        raise InvalidAccountId(f"'{value}' is not a valid {field}.")


def validate_create_account(body):
    _require_object(body)
    errors = []
    _reject_unknown(body, {"name"}, errors)
    name = _string(body, "name", errors, required=True, max_len=MAX_NAME)
    _finish(errors)
    return {"name": name}


def validate_create_transaction(body):
    _require_object(body)
    errors = []
    _reject_unknown(body, {"type", "amount", "description", "reference_id"}, errors)

    tx_type = body.get("type")
    if tx_type is None:
        errors.append({"field": "type", "code": "REQUIRED", "message": "type is required."})
    elif not isinstance(tx_type, str) or tx_type not in TRANSACTION_TYPES:
        errors.append({"field": "type", "code": "INVALID_TRANSACTION_TYPE",
                       "message": "type must be either 'credit' or 'debit'."})
        tx_type = None

    amount = _amount(body, errors)
    description = _string(body, "description", errors, required=False, max_len=MAX_DESCRIPTION)
    reference_id = _string(body, "reference_id", errors, required=True, max_len=MAX_REFERENCE)
    _finish(errors)
    return {"type": tx_type, "amount_minor": amount,
            "description": description or "", "reference_id": reference_id}


def validate_create_transfer(body):
    _require_object(body)
    errors = []
    _reject_unknown(body, {"from_account_id", "to_account_id", "amount",
                           "description", "reference_id"}, errors)
    ids = {}
    for field in ("from_account_id", "to_account_id"):
        raw = body.get(field)
        if raw is None:
            errors.append({"field": field, "code": "REQUIRED", "message": f"{field} is required."})
            continue
        try:
            ids[field] = str(uuid.UUID(raw)) if isinstance(raw, str) else None
        except ValueError:
            ids[field] = None
        if ids[field] is None:
            errors.append({"field": field, "code": "INVALID_ACCOUNT_ID",
                           "message": f"{field} must be a valid account id."})
    if (len(ids) == 2 and None not in ids.values()
            and ids["from_account_id"] == ids["to_account_id"]):
        errors.append({"field": "to_account_id", "code": "SAME_ACCOUNT",
                       "message": "Source and destination accounts must be different."})

    amount = _amount(body, errors)
    description = _string(body, "description", errors, required=False, max_len=MAX_DESCRIPTION)
    reference_id = _string(body, "reference_id", errors, required=True, max_len=MAX_REFERENCE)
    _finish(errors)
    return {"from_account_id": ids["from_account_id"], "to_account_id": ids["to_account_id"],
            "amount_minor": amount, "description": description or "", "reference_id": reference_id}


def _positive_int(raw, name, default, maximum=None):
    if raw is None:
        return default
    if not raw.isascii() or not raw.isdigit() or int(raw) < 1:
        raise InvalidPagination(
            f"'{name}' must be a positive integer.",
            [{"field": name, "code": "INVALID_PAGINATION", "message": f"{name} must be >= 1."}])
    value = int(raw)
    if maximum and value > maximum:
        raise InvalidPagination(
            f"'{name}' must be at most {maximum}.",
            [{"field": name, "code": "INVALID_PAGINATION", "message": f"{name} must be <= {maximum}."}])
    return value


def parse_pagination(args):
    return (_positive_int(args.get("page"), "page", 1),
            _positive_int(args.get("limit"), "limit", DEFAULT_LIMIT, MAX_LIMIT))
```

app/services.py

```
"""Business logic. Routes stay thin; every money movement goes through here."""
import math
import sqlite3
import uuid
from datetime import datetime, timezone

from .db import read_snapshot, write_transaction
from .errors import (AccountNotFound, DuplicateTransaction, InsufficientBalance,
                     TransactionNotFound)
from .money import format_minor


def _now():
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%fZ")


def _account_dict(row):
    return {"id": row["id"], "name": row["name"],
            "balance": format_minor(row["balance_minor"]), "created_at": row["created_at"]}


def _transaction_dict(row):
    return {"id": row["id"], "account_id": row["account_id"], "type": row["type"],
            "amount": format_minor(row["amount_minor"]), "description": row["description"],
            "reference_id": row["reference_id"], "transfer_id": row["transfer_id"],
            "created_at": row["created_at"]}


def _transfer_dict(row):
    return {"id": row["id"], "from_account_id": row["from_account_id"],
            "to_account_id": row["to_account_id"], "amount": format_minor(row["amount_minor"]),
            "description": row["description"], "reference_id": row["reference_id"],
            "created_at": row["created_at"]}


def _paginated(items, page, limit, total):
    return {"data": items,
            "pagination": {"page": page, "limit": limit, "total_records": total,
                           "total_pages": math.ceil(total / limit)}}


# ---------------------------------------------------------------- accounts

def create_account(db, name):
    account_id, now = str(uuid.uuid4()), _now()
    with write_transaction(db):
        # balance is never taken from the client: it always starts at 0.
        db.execute("INSERT INTO accounts (id, name, balance_minor, created_at) VALUES (?, ?, 0, ?)",
                   (account_id, name, now))
    return get_account(db, account_id)


def _fetch_account(db, account_id, field="account_id"):
    row = db.execute("SELECT * FROM accounts WHERE id = ?", (account_id,)).fetchone()
    if row is None:
        raise AccountNotFound(f"Account '{account_id}' was not found.",
                              [{"field": field, "code": "ACCOUNT_NOT_FOUND",
                                "message": "No account with this id."}])
    return row


def get_account(db, account_id):
    return _account_dict(_fetch_account(db, account_id))


def list_accounts(db, page, limit):
    with read_snapshot(db):
        total = db.execute("SELECT COUNT(*) FROM accounts").fetchone()[0]
        rows = db.execute("SELECT * FROM accounts ORDER BY created_at DESC, rowid DESC "
                          "LIMIT ? OFFSET ?", (limit, (page - 1) * limit)).fetchall()
    return _paginated([_account_dict(r) for r in rows], page, limit, total)


# ------------------------------------------------------------ transactions

def _apply_balance(db, account_id, delta_minor):
    cur = db.execute("UPDATE accounts SET balance_minor = balance_minor + ? WHERE id = ?",
                     (delta_minor, account_id))
    if cur.rowcount != 1:
        raise RuntimeError("balance update affected an unexpected number of rows")


def _post_entry(db, account_id, tx_type, amount_minor, description, reference_id,
                transfer_id=None, field="account_id"):
    """Insert one ledger entry and move the balance. MUST run inside write_transaction:
    any exception raised here rolls back both the insert and the balance change."""
    account = _fetch_account(db, account_id, field)
    if tx_type == "debit" and account["balance_minor"] < amount_minor:
        raise InsufficientBalance(
            "Insufficient balance for this debit.",
            [{"field": "amount", "code": "INSUFFICIENT_BALANCE",
              "message": f"Available balance is {format_minor(account['balance_minor'])}, "
                         f"requested {format_minor(amount_minor)}."}])

    tx_id = str(uuid.uuid4())
    try:
        db.execute(
            "INSERT INTO transactions (id, account_id, type, amount_minor, description, "
            "reference_id, transfer_id, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
            (tx_id, account_id, tx_type, amount_minor, description, reference_id,
             transfer_id, _now()))
    except sqlite3.IntegrityError as exc:
        if "UNIQUE" in str(exc):
            raise DuplicateTransaction(
                f"A transaction with reference_id '{reference_id}' already exists for this account.",
                [{"field": "reference_id", "code": "DUPLICATE_REFERENCE",
                  "message": "reference_id must be unique within an account."}])
        raise
    _apply_balance(db, account_id, amount_minor if tx_type == "credit" else -amount_minor)
    return tx_id


def add_transaction(db, account_id, tx_type, amount_minor, description, reference_id):
    with write_transaction(db):
        tx_id = _post_entry(db, account_id, tx_type, amount_minor, description, reference_id)
    return {"transaction": _transaction_dict(
                db.execute("SELECT * FROM transactions WHERE id = ?", (tx_id,)).fetchone()),
            "account": get_account(db, account_id)}


def get_transaction(db, account_id, tx_id):
    _fetch_account(db, account_id)
    row = db.execute("SELECT * FROM transactions WHERE id = ? AND account_id = ?",
                     (tx_id, account_id)).fetchone()
    if row is None:
        raise TransactionNotFound(f"Transaction '{tx_id}' was not found for this account.")
    return _transaction_dict(row)


def list_transactions(db, account_id, page, limit):
    with read_snapshot(db):
        _fetch_account(db, account_id)
        total = db.execute("SELECT COUNT(*) FROM transactions WHERE account_id = ?",
                           (account_id,)).fetchone()[0]
        rows = db.execute(
            "SELECT * FROM transactions WHERE account_id = ? "
            "ORDER BY created_at DESC, rowid DESC LIMIT ? OFFSET ?",
            (account_id, limit, (page - 1) * limit)).fetchall()
    return _paginated([_transaction_dict(r) for r in rows], page, limit, total)


# --------------------------------------------------------------- transfers

def create_transfer(db, from_id, to_id, amount_minor, description, reference_id):
    transfer_id, now = str(uuid.uuid4()), _now()
    with write_transaction(db):
        _fetch_account(db, from_id, "from_account_id")
        _fetch_account(db, to_id, "to_account_id")
        try:
            db.execute(
                "INSERT INTO transfers (id, from_account_id, to_account_id, amount_minor, "
                "reference_id, description, created_at) VALUES (?, ?, ?, ?, ?, ?, ?)",
                (transfer_id, from_id, to_id, amount_minor, reference_id, description, now))
        except sqlite3.IntegrityError as exc:
            if "UNIQUE" in str(exc):
                raise DuplicateTransaction(
                    f"A transfer with reference_id '{reference_id}' already exists "
                    "for the source account.",
                    [{"field": "reference_id", "code": "DUPLICATE_REFERENCE",
                      "message": "reference_id must be unique per source account."}])
            raise
        # Debit first: if the source lacks funds, nothing has been written yet,
        # and anything raised later rolls back the whole transfer.
        _post_entry(db, from_id, "debit", amount_minor,
                    description or f"Transfer to account {to_id}",
                    f"transfer:{transfer_id}:debit", transfer_id, "from_account_id")
        _post_entry(db, to_id, "credit", amount_minor,
                    description or f"Transfer from account {from_id}",
                    f"transfer:{transfer_id}:credit", transfer_id, "to_account_id")
    transfer = db.execute("SELECT * FROM transfers WHERE id = ?", (transfer_id,)).fetchone()
    return {"transfer": _transfer_dict(transfer),
            "from_account": get_account(db, from_id), "to_account": get_account(db, to_id)}
```

app/db.py

```
"""SQLite access. One connection per request; explicit transaction control."""
import sqlite3
from contextlib import contextmanager
from pathlib import Path

from flask import current_app, g

SCHEMA = Path(__file__).with_name("schema.sql")


def connect(path):
    # isolation_level=None -> we issue BEGIN/COMMIT/ROLLBACK ourselves.
    conn = sqlite3.connect(path, isolation_level=None, timeout=10)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA busy_timeout = 10000")
    return conn


def get_db():
    if "db" not in g:
        g.db = connect(current_app.config["DATABASE"])
    return g.db


def close_db(_exc=None):
    db = g.pop("db", None)
    if db is not None:
        db.close()


def init_db(path):
    conn = connect(path)
    try:
        conn.executescript(SCHEMA.read_text())
    finally:
        conn.close()


@contextmanager
def write_transaction(db):
    """All-or-nothing write. BEGIN IMMEDIATE takes the write lock up front, so
    concurrent writers are serialised and balance checks cannot race."""
    db.execute("BEGIN IMMEDIATE")
    try:
        yield db
    except BaseException:
        db.execute("ROLLBACK")
        raise
    else:
        try:
            db.execute("COMMIT")
        except BaseException:
            db.execute("ROLLBACK")
            raise


@contextmanager
def read_snapshot(db):
    """Consistent view across multiple SELECTs (e.g. COUNT + page of rows)."""
    db.execute("BEGIN")
    try:
        yield db
    finally:
        db.execute("COMMIT")
```

app/money.py

```
"""Money helpers. Amounts are exchanged as decimals, stored as integer cents."""
from decimal import Decimal, InvalidOperation

MAX_AMOUNT_MINOR = 1_000_000_000 * 100  # 1,000,000,000.00 per transaction


def parse_amount(value):
    """Return amount in minor units, or raise ValueError(message)."""
    if isinstance(value, bool) or not isinstance(value, (int, float, str)):
        raise ValueError("amount must be a number or a numeric string")
    try:
        d = Decimal(str(value).strip())
    except InvalidOperation:
        raise ValueError("amount must be a valid number")
    if not d.is_finite():
        raise ValueError("amount must be a finite number")
    if d <= 0:
        raise ValueError("amount must be greater than 0")
    minor = d * 100
    if minor != minor.to_integral_value():
        raise ValueError("amount can have at most 2 decimal places")
    minor = int(minor)
    if minor > MAX_AMOUNT_MINOR:
        raise ValueError("amount exceeds the maximum allowed per transaction")
    return minor


def format_minor(minor):
    """12345 -> '123.45' (always 2 decimals, returned as a string)."""
    return f"{minor // 100}.{minor % 100:02d}"
```

app/errors.py

```
"""Application errors and the single, consistent error response shape.

    {"error": {"code": "...", "message": "...", "details": [...]}}
"""
from flask import jsonify


class ApiError(Exception):
    status = 500
    code = "INTERNAL_ERROR"

    def __init__(self, message, details=None):
        super().__init__(message)
        self.message = message
        self.details = details or []


class InvalidRequestBody(ApiError):
    status = 400
    code = "INVALID_REQUEST_BODY"


class InvalidPagination(ApiError):
    status = 400
    code = "INVALID_PAGINATION"


class InvalidAccountId(ApiError):
    status = 400
    code = "INVALID_ACCOUNT_ID"


class ValidationError(ApiError):
    status = 422
    code = "VALIDATION_ERROR"


class AccountNotFound(ApiError):
    status = 404
    code = "ACCOUNT_NOT_FOUND"


class TransactionNotFound(ApiError):
    status = 404
    code = "TRANSACTION_NOT_FOUND"


class InsufficientBalance(ApiError):
    status = 422
    code = "INSUFFICIENT_BALANCE"


class DuplicateTransaction(ApiError):
    status = 409
    code = "DUPLICATE_TRANSACTION"


def error_body(code, message, details=None):
    return {"error": {"code": code, "message": message, "details": details or []}}


def register_error_handlers(app):
    @app.errorhandler(ApiError)
    def handle_api_error(exc):
        return jsonify(error_body(exc.code, exc.message, exc.details)), exc.status

    @app.errorhandler(404)
    def handle_404(_exc):
        return jsonify(error_body("NOT_FOUND", "The requested resource does not exist.")), 404

    @app.errorhandler(405)
    def handle_405(exc):
        response = jsonify(
            error_body("METHOD_NOT_ALLOWED", "This method is not allowed for this resource.")
        )
        response.status_code = 405
        if exc.valid_methods:
            response.headers["Allow"] = ", ".join(sorted(exc.valid_methods))
        return response

    @app.errorhandler(Exception)
    def handle_unexpected(exc):
        app.logger.exception("Unhandled error: %s", exc)
        return jsonify(error_body("INTERNAL_ERROR", "An unexpected error occurred.")), 500
```

app/schema.sql

```
-- Money is stored as INTEGER minor units (cents) to avoid floating point error.

CREATE TABLE IF NOT EXISTS accounts (
    id            TEXT PRIMARY KEY,
    name          TEXT NOT NULL CHECK (length(trim(name)) > 0),
    balance_minor INTEGER NOT NULL DEFAULT 0 CHECK (balance_minor >= 0),
    created_at    TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS transfers (
    id                TEXT PRIMARY KEY,
    from_account_id   TEXT NOT NULL REFERENCES accounts(id),
    to_account_id     TEXT NOT NULL REFERENCES accounts(id),
    amount_minor      INTEGER NOT NULL CHECK (amount_minor > 0),
    reference_id      TEXT NOT NULL CHECK (length(trim(reference_id)) > 0),
    description       TEXT NOT NULL DEFAULT '',
    created_at        TEXT NOT NULL,
    CHECK (from_account_id <> to_account_id),
    UNIQUE (from_account_id, reference_id)
);

CREATE TABLE IF NOT EXISTS transactions (
    id            TEXT PRIMARY KEY,
    account_id    TEXT NOT NULL REFERENCES accounts(id),
    type          TEXT NOT NULL CHECK (type IN ('credit', 'debit')),
    amount_minor  INTEGER NOT NULL CHECK (amount_minor > 0),
    description   TEXT NOT NULL DEFAULT '',
    reference_id  TEXT NOT NULL CHECK (length(trim(reference_id)) > 0),
    transfer_id   TEXT REFERENCES transfers(id),
    created_at    TEXT NOT NULL,
    -- reference_id is unique *within* an account (not globally)
    UNIQUE (account_id, reference_id)
);

CREATE INDEX IF NOT EXISTS idx_transactions_account_created
    ON transactions (account_id, created_at DESC);

-- Immutability enforced at the database level, not only by the API surface.
CREATE TRIGGER IF NOT EXISTS transactions_no_update
BEFORE UPDATE ON transactions
BEGIN
    SELECT RAISE(ABORT, 'transactions are immutable');
END;

CREATE TRIGGER IF NOT EXISTS transactions_no_delete
BEFORE DELETE ON transactions
BEGIN
    SELECT RAISE(ABORT, 'transactions are immutable');
END;

CREATE TRIGGER IF NOT EXISTS transfers_no_update
BEFORE UPDATE ON transfers
BEGIN
    SELECT RAISE(ABORT, 'transfers are immutable');
END;

CREATE TRIGGER IF NOT EXISTS transfers_no_delete
BEFORE DELETE ON transfers
BEGIN
    SELECT RAISE(ABORT, 'transfers are immutable');
END;
```

run.py

```
"""Entry point: `python run.py` starts the dev server on http://127.0.0.1:5000."""
import os

from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(
        host=os.environ.get("HOST", "127.0.0.1"),
        port=int(os.environ.get("PORT", "5000")),
        debug=os.environ.get("FLASK_DEBUG") == "1",
    )
```

requirements.txt

```
Flask>=3.0
```

tests/test_api.py

```
import os
import sqlite3
import tempfile
import threading
import unittest
from unittest import mock

from app import create_app, services


class ApiTestCase(unittest.TestCase):
    def setUp(self):
        fd, self.db_path = tempfile.mkstemp(suffix=".db")
        os.close(fd)
        self.app = create_app({"DATABASE": self.db_path, "TESTING": True})
        self.client = self.app.test_client()

    def tearDown(self):
        for suffix in ("", "-wal", "-shm"):
            try:
                os.remove(self.db_path + suffix)
            except FileNotFoundError:
                pass

    # helpers
    def account(self, name="Alice"):
        r = self.client.post("/accounts", json={"name": name})
        self.assertEqual(r.status_code, 201, r.get_json())
        return r.get_json()["id"]

    def tx(self, account_id, type_, amount, ref, client=None, **extra):
        body = {"type": type_, "amount": amount, "reference_id": ref, **extra}
        return (client or self.client).post(f"/accounts/{account_id}/transactions", json=body)

    def balance(self, account_id):
        return self.client.get(f"/accounts/{account_id}").get_json()["balance"]

    def assert_error(self, resp, status, code):
        self.assertEqual(resp.status_code, status, resp.get_json())
        err = resp.get_json()["error"]
        self.assertEqual(err["code"], code)
        self.assertIn("message", err)
        self.assertIn("details", err)
        return err

    def ledger_matches_balance(self, account_id):
        conn = sqlite3.connect(self.db_path)
        credits, debits = conn.execute(
            "SELECT COALESCE(SUM(CASE WHEN type='credit' THEN amount_minor END),0), "
            "COALESCE(SUM(CASE WHEN type='debit' THEN amount_minor END),0) "
            "FROM transactions WHERE account_id=?", (account_id,)).fetchone()
        bal = conn.execute("SELECT balance_minor FROM accounts WHERE id=?", (account_id,)).fetchone()[0]
        conn.close()
        self.assertEqual(bal, credits - debits)


class AccountTests(ApiTestCase):
    def test_create_account(self):
        r = self.client.post("/accounts", json={"name": "  Alice "})
        self.assertEqual(r.status_code, 201)
        body = r.get_json()
        self.assertEqual(body["name"], "Alice")
        self.assertEqual(body["balance"], "0.00")
        self.assertTrue(body["id"] and body["created_at"])
        self.assertEqual(self.client.get(f"/accounts/{body['id']}").get_json(), body)

    def test_ids_are_unique(self):
        self.assertNotEqual(self.account(), self.account())

    def test_client_cannot_set_balance_or_system_fields(self):
        for field, value in (("balance", 500), ("id", "x"), ("created_at", "2000-01-01")):
            r = self.client.post("/accounts", json={"name": "A", field: value})
            err = self.assert_error(r, 422, "VALIDATION_ERROR")
            self.assertEqual(err["details"][0]["field"], field)

    def test_name_required(self):
        for body in ({}, {"name": ""}, {"name": "   "}, {"name": None}):
            self.assert_error(self.client.post("/accounts", json=body), 422, "VALIDATION_ERROR")
        self.assert_error(self.client.post("/accounts", json={"name": 5}), 422, "VALIDATION_ERROR")

    def test_invalid_body(self):
        self.assert_error(self.client.post("/accounts", data="nope", content_type="application/json"),
                          400, "INVALID_REQUEST_BODY")
        self.assert_error(self.client.post("/accounts", json=["a"]), 400, "INVALID_REQUEST_BODY")
        self.assert_error(self.client.post("/accounts"), 400, "INVALID_REQUEST_BODY")

    def test_get_account_errors(self):
        self.assert_error(self.client.get("/accounts/not-a-uuid"), 400, "INVALID_ACCOUNT_ID")
        self.assert_error(self.client.get("/accounts/6f1c0f0e-1111-4222-8333-944455556666"),
                          404, "ACCOUNT_NOT_FOUND")


class TransactionTests(ApiTestCase):
    def test_credit_and_debit_sequence(self):
        a = self.account()
        for i, (t, amt) in enumerate([("credit", 100), ("credit", 50), ("debit", 30), ("debit", 20)]):
            r = self.tx(a, t, amt, f"r{i}")
            self.assertEqual(r.status_code, 201, r.get_json())
        self.assertEqual(self.balance(a), "100.00")
        self.ledger_matches_balance(a)

    def test_credit_example_100_plus_50(self):
        a = self.account()
        self.tx(a, "credit", 100, "a")
        r = self.tx(a, "credit", 50, "b")
        self.assertEqual(r.get_json()["account"]["balance"], "150.00")

    def test_response_shape(self):
        a = self.account()
        t = self.tx(a, "credit", "10.50", "ref-1", description="salary").get_json()["transaction"]
        self.assertEqual((t["account_id"], t["type"], t["amount"], t["description"], t["reference_id"]),
                         (a, "credit", "10.50", "salary", "ref-1"))
        self.assertTrue(t["id"] and t["created_at"])

    def test_decimal_precision(self):
        a = self.account()
        for i in range(10):
            self.tx(a, "credit", 0.1, f"r{i}")
        self.assertEqual(self.balance(a), "1.00")  # no float drift

    def test_insufficient_balance_rejected_and_no_effect(self):
        a = self.account()
        self.tx(a, "credit", 100, "c")
        self.assert_error(self.tx(a, "debit", "100.01", "d"), 422, "INSUFFICIENT_BALANCE")
        self.assertEqual(self.balance(a), "100.00")
        self.assertEqual(self.client.get(f"/accounts/{a}/transactions").get_json()
                         ["pagination"]["total_records"], 1)
        self.assertEqual(self.tx(a, "debit", 100, "d").status_code, 201)  # exact balance is allowed
        self.assertEqual(self.balance(a), "0.00")

    def test_duplicate_reference_same_account(self):
        a = self.account()
        self.tx(a, "credit", 100, "dup")
        self.assert_error(self.tx(a, "credit", 100, "dup"), 409, "DUPLICATE_TRANSACTION")
        self.assert_error(self.tx(a, "debit", 1, "dup"), 409, "DUPLICATE_TRANSACTION")
        self.assertEqual(self.balance(a), "100.00")

    def test_same_reference_allowed_on_different_accounts(self):
        a, b = self.account("A"), self.account("B")
        self.assertEqual(self.tx(a, "credit", 5, "same").status_code, 201)
        self.assertEqual(self.tx(b, "credit", 5, "same").status_code, 201)

    def test_validation_errors(self):
        a = self.account()
        self.assert_error(self.tx(a, "refund", 5, "r"), 422, "VALIDATION_ERROR")
        self.assert_error(self.tx(a, "CREDIT", 5, "r"), 422, "VALIDATION_ERROR")
        self.assertEqual(self.tx(a, "refund", 5, "r").get_json()["error"]["details"][0]["code"],
                         "INVALID_TRANSACTION_TYPE")
        bad_amounts = [0, -5, "abc", None, True, [], "1.234", "NaN", "Infinity", 1e30, ""]
        for i, amt in enumerate(bad_amounts):
            r = self.tx(a, "credit", amt, f"x{i}")
            self.assert_error(r, 422, "VALIDATION_ERROR")
        self.assertEqual(self.tx(a, "credit", -5, "q").get_json()["error"]["details"][0]["code"],
                         "INVALID_AMOUNT")
        r = self.client.post(f"/accounts/{a}/transactions", json={"type": "credit", "amount": 5})
        self.assertEqual(r.get_json()["error"]["details"][0]["field"], "reference_id")
        self.assert_error(self.tx(a, "credit", 5, "   "), 422, "VALIDATION_ERROR")
        self.assert_error(self.tx(a, "credit", 5, "ok", description=5), 422, "VALIDATION_ERROR")
        self.assertEqual(self.balance(a), "0.00")

    def test_multiple_errors_reported_together(self):
        a = self.account()
        r = self.client.post(f"/accounts/{a}/transactions", json={})
        fields = {d["field"] for d in r.get_json()["error"]["details"]}
        self.assertEqual(fields, {"type", "amount", "reference_id"})

    def test_account_errors(self):
        self.assert_error(self.tx("bad-id", "credit", 5, "r"), 400, "INVALID_ACCOUNT_ID")
        self.assert_error(self.tx("6f1c0f0e-1111-4222-8333-944455556666", "credit", 5, "r"),
                          404, "ACCOUNT_NOT_FOUND")

    def test_invalid_body(self):
        a = self.account()
        r = self.client.post(f"/accounts/{a}/transactions", data="{bad", content_type="application/json")
        self.assert_error(r, 400, "INVALID_REQUEST_BODY")

    def test_get_single_transaction(self):
        a, b = self.account("A"), self.account("B")
        tid = self.tx(a, "credit", 5, "r").get_json()["transaction"]["id"]
        self.assertEqual(self.client.get(f"/accounts/{a}/transactions/{tid}").status_code, 200)
        self.assert_error(self.client.get(f"/accounts/{b}/transactions/{tid}"), 404, "TRANSACTION_NOT_FOUND")


class ListingTests(ApiTestCase):
    def test_latest_first_and_pagination(self):
        a = self.account()
        for i in range(25):
            self.tx(a, "credit", 1, f"r{i:02d}")
        p1 = self.client.get(f"/accounts/{a}/transactions?page=1&limit=10").get_json()
        self.assertEqual(p1["pagination"], {"page": 1, "limit": 10, "total_records": 25, "total_pages": 3})
        self.assertEqual([t["reference_id"] for t in p1["data"]], [f"r{i:02d}" for i in range(24, 14, -1)])
        p3 = self.client.get(f"/accounts/{a}/transactions?page=3&limit=10").get_json()
        self.assertEqual([t["reference_id"] for t in p3["data"]], [f"r{i:02d}" for i in range(4, -1, -1)])
        stamps = [t["created_at"] for t in p1["data"]]
        self.assertEqual(stamps, sorted(stamps, reverse=True))

    def test_defaults_and_empty(self):
        a = self.account()
        r = self.client.get(f"/accounts/{a}/transactions").get_json()
        self.assertEqual(r["data"], [])
        self.assertEqual(r["pagination"], {"page": 1, "limit": 20, "total_records": 0, "total_pages": 0})

    def test_page_past_end_is_empty(self):
        a = self.account()
        self.tx(a, "credit", 1, "r")
        r = self.client.get(f"/accounts/{a}/transactions?page=9&limit=10").get_json()
        self.assertEqual(r["data"], [])
        self.assertEqual(r["pagination"]["total_pages"], 1)

    def test_invalid_pagination(self):
        a = self.account()
        for q in ("page=0", "page=-1", "page=abc", "limit=0", "limit=101", "limit=1.5", "page="):
            self.assert_error(self.client.get(f"/accounts/{a}/transactions?{q}"), 400, "INVALID_PAGINATION")

    def test_listing_unknown_account(self):
        self.assert_error(self.client.get("/accounts/6f1c0f0e-1111-4222-8333-944455556666/transactions"),
                          404, "ACCOUNT_NOT_FOUND")

    def test_list_accounts(self):
        for n in "ABC":
            self.account(n)
        r = self.client.get("/accounts?limit=2").get_json()
        self.assertEqual([x["name"] for x in r["data"]], ["C", "B"])
        self.assertEqual(r["pagination"]["total_pages"], 2)


class ImmutabilityTests(ApiTestCase):
    def test_no_api_for_modifying_transactions(self):
        a = self.account()
        tid = self.tx(a, "credit", 5, "r").get_json()["transaction"]["id"]
        url = f"/accounts/{a}/transactions/{tid}"
        for method in ("put", "patch", "delete"):
            r = getattr(self.client, method)(url, json={"amount": 1})
            self.assert_error(r, 405, "METHOD_NOT_ALLOWED")
        self.assert_error(self.client.delete(f"/accounts/{a}/transactions"), 405, "METHOD_NOT_ALLOWED")
        self.assertEqual(self.client.get(url).get_json()["amount"], "5.00")

    def test_database_blocks_update_and_delete(self):
        a = self.account()
        self.tx(a, "credit", 5, "r")
        conn = sqlite3.connect(self.db_path)
        with self.assertRaises(sqlite3.DatabaseError):
            conn.execute("UPDATE transactions SET amount_minor = 999")
        with self.assertRaises(sqlite3.DatabaseError):
            conn.execute("DELETE FROM transactions")
        conn.close()


class AtomicityTests(ApiTestCase):
    def test_failure_after_insert_rolls_back_everything(self):
        a = self.account()
        self.tx(a, "credit", 100, "seed")
        with mock.patch.object(services, "_apply_balance", side_effect=RuntimeError("boom")):
            r = self.tx(a, "credit", 50, "will-fail")
        self.assert_error(r, 500, "INTERNAL_ERROR")
        self.assertEqual(self.balance(a), "100.00")
        refs = [t["reference_id"] for t in
                self.client.get(f"/accounts/{a}/transactions").get_json()["data"]]
        self.assertEqual(refs, ["seed"])
        # the reference is not "burned" by the failed attempt
        self.assertEqual(self.tx(a, "credit", 50, "will-fail").status_code, 201)
        self.ledger_matches_balance(a)

    def test_concurrent_debits_never_overdraw(self):
        a = self.account()
        self.tx(a, "credit", 100, "seed")
        results, lock = [], threading.Lock()

        def worker(i):
            c = self.app.test_client()
            code = self.tx(a, "debit", 10, f"d{i}", client=c).status_code
            with lock:
                results.append(code)

        threads = [threading.Thread(target=worker, args=(i,)) for i in range(25)]
        [t.start() for t in threads]
        [t.join() for t in threads]
        self.assertEqual(results.count(201), 10)
        self.assertEqual(results.count(422), 15)
        self.assertEqual(self.balance(a), "0.00")
        self.ledger_matches_balance(a)


class TransferTests(ApiTestCase):
    def transfer(self, src, dst, amount, ref="t1", **extra):
        return self.client.post("/transfers", json={
            "from_account_id": src, "to_account_id": dst, "amount": amount,
            "reference_id": ref, **extra})

    def test_successful_transfer(self):
        a, b = self.account("A"), self.account("B")
        self.tx(a, "credit", 100, "s1")
        self.tx(b, "credit", 50, "s2")
        r = self.transfer(a, b, 30)
        self.assertEqual(r.status_code, 201, r.get_json())
        body = r.get_json()
        self.assertEqual(body["from_account"]["balance"], "70.00")
        self.assertEqual(body["to_account"]["balance"], "80.00")
        self.assertEqual(self.balance(a), "70.00")
        self.assertEqual(self.balance(b), "80.00")
        da = self.client.get(f"/accounts/{a}/transactions").get_json()["data"][0]
        db_ = self.client.get(f"/accounts/{b}/transactions").get_json()["data"][0]
        self.assertEqual((da["type"], db_["type"]), ("debit", "credit"))
        self.assertEqual(da["transfer_id"], db_["transfer_id"])
        self.ledger_matches_balance(a)
        self.ledger_matches_balance(b)

    def test_insufficient_balance(self):
        a, b = self.account("A"), self.account("B")
        self.tx(a, "credit", 10, "s")
        self.assert_error(self.transfer(a, b, 10.01), 422, "INSUFFICIENT_BALANCE")
        self.assertEqual((self.balance(a), self.balance(b)), ("10.00", "0.00"))

    def test_missing_accounts(self):
        a = self.account()
        ghost = "6f1c0f0e-1111-4222-8333-944455556666"
        for src, dst, field in ((a, ghost, "to_account_id"), (ghost, a, "from_account_id")):
            err = self.assert_error(self.transfer(src, dst, 1), 404, "ACCOUNT_NOT_FOUND")
            self.assertEqual(err["details"][0]["field"], field)

    def test_validation(self):
        a, b = self.account("A"), self.account("B")
        self.tx(a, "credit", 100, "s")
        self.assert_error(self.transfer(a, a, 5), 422, "VALIDATION_ERROR")
        self.assert_error(self.transfer(a, b, 0), 422, "VALIDATION_ERROR")
        self.assert_error(self.transfer("nope", b, 5), 422, "VALIDATION_ERROR")
        self.assert_error(self.client.post("/transfers", json={}), 422, "VALIDATION_ERROR")
        self.assert_error(self.client.post("/transfers", data="x", content_type="application/json"),
                          400, "INVALID_REQUEST_BODY")
        self.assertEqual(self.balance(a), "100.00")

    def test_duplicate_transfer_reference(self):
        a, b = self.account("A"), self.account("B")
        self.tx(a, "credit", 100, "s")
        self.assertEqual(self.transfer(a, b, 10, "same").status_code, 201)
        self.assert_error(self.transfer(a, b, 10, "same"), 409, "DUPLICATE_TRANSACTION")
        self.assertEqual((self.balance(a), self.balance(b)), ("90.00", "10.00"))

    def test_failure_on_credit_leg_rolls_back_debit_leg(self):
        a, b = self.account("A"), self.account("B")
        self.tx(a, "credit", 100, "s")
        real = services._apply_balance
        calls = {"n": 0}

        def flaky(db, account_id, delta):
            calls["n"] += 1
            if calls["n"] == 2:  # second leg (credit to B)
                raise RuntimeError("boom")
            return real(db, account_id, delta)

        with mock.patch.object(services, "_apply_balance", side_effect=flaky):
            r = self.transfer(a, b, 40)
        self.assert_error(r, 500, "INTERNAL_ERROR")
        self.assertEqual((self.balance(a), self.balance(b)), ("100.00", "0.00"))
        conn = sqlite3.connect(self.db_path)
        self.assertEqual(conn.execute("SELECT COUNT(*) FROM transfers").fetchone()[0], 0)
        self.assertEqual(conn.execute("SELECT COUNT(*) FROM transactions").fetchone()[0], 1)
        conn.close()


class MiscTests(ApiTestCase):
    def test_unknown_route_json_404(self):
        self.assert_error(self.client.get("/nope"), 404, "NOT_FOUND")

    def test_health(self):
        self.assertEqual(self.client.get("/health").get_json(), {"status": "ok"})


if __name__ == "__main__":
    unittest.main()
```