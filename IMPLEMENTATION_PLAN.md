# TaxOS CA Firm Transformation — Implementation Plan

Transform TaxOS from a single-freelancer edge hub into a **CA firm practice management suite** with per-taxpayer isolated ledgers, batch compliance tooling, and notice defense workflows.

---

## Architecture Target

```
CA Firm Portal (/ca)
    ├── taxpayers (roster) ──each──► users (isolated ledger)
    │                                    ├── clients
    │                                    ├── invoices
    │                                    └── approvals
    ├── staff_members (RBAC — future)
    ├── notice_tracker
    └── ais_entries (scoped by taxpayer_id)
```

---

## Phase 1 — Per-Taxpayer Data Isolation ✅ COMPLETE

**Goal:** Each CA client gets their own ledger; bulk AIS and advance tax operate on scoped data, not the global freelancer pool.

### 1.1 Schema migrations ✅

| Change | Table | Column | Status |
|--------|-------|--------|--------|
| Link taxpayer → ledger | `taxpayers` | `user_id INTEGER FK → users` | Done |
| Scope AIS per client | `ais_entries` | `taxpayer_id INTEGER FK → taxpayers` | Done |

Migrations applied via `ALTER TABLE` in `database.init_db()` for existing databases.

### 1.2 `add_taxpayer` creates isolated ledger ✅

When a CA firm adds a client taxpayer, the system now:

1. Creates a `users` row (encrypted name, PAN, email)
2. Creates a default `clients` row ("Primary Client")
3. Seeds a genesis `approvals` block for that user
4. Stores `user_id` on the `taxpayers` record

**File:** `database.py` — `add_taxpayer()`, `get_primary_client_id()`

### 1.3 Scoped invoice queries ✅

`get_invoices(include_inactive=False, user_id=None)` accepts an optional `user_id` filter so each taxpayer's books are queried independently.

**File:** `database.py`

### 1.4 Scoped AIS storage ✅

| Function | Behavior |
|----------|----------|
| `add_ais_entry(..., taxpayer_id=None)` | `NULL` = freelancer global pool; integer = CA client scope |
| `get_ais_entries(taxpayer_id=None)` | Returns entries for that scope only |
| `clear_ais_entries(taxpayer_id=None)` | Clears only the targeted scope (no cross-client wipe) |

**File:** `database.py`

Freelancer AIS import (`POST /ais/import`) unchanged — uses `taxpayer_id=NULL` pool.

### 1.5 Bulk AIS reconciliation fix ✅

`ca_engine.bulk_ais_reconcile()` now:

- Clears AIS entries **per taxpayer** (not globally)
- Loads invoices via `get_invoices(user_id=taxpayer.user_id)`
- Passes scoped data into `ais_engine.reconcile(invoices=..., ais_entries=...)`
- Detects anomalies from `unmatched_in_ais` and `variance_inr` (fixed incorrect key names)

**File:** `ca_engine.py`

### 1.6 Advance tax revenue fix ✅

`ca_engine.batch_advance_tax_calculator()` now sums invoices from each taxpayer's `user_id` ledger instead of incorrectly mapping `invoice.client_id → taxpayer.id`.

Fallback of ₹18,00,000 retained when a taxpayer has no invoices yet.

**File:** `ca_engine.py`

### 1.7 API update ✅

`POST /api/ca/taxpayers` now returns `{ status, taxpayer_id, user_id }`.

**File:** `main.py`

### 1.8 Tests ✅

`test_ca_firm.py` extended with:

- `test_taxpayer_gets_isolated_ledger` — distinct `user_id` per taxpayer
- `test_bulk_ais_reconciliation` — t1 compliant, t2 flagged (with per-ledger invoices)
- `test_batch_advance_tax_calculator` — revenue from scoped invoices (₹30L vs ₹18L fallback)

---

## Phase 2 — API + UI Completeness ⬜ NOT STARTED

| Task | Status |
|------|--------|
| `GET /api/ca/notices` | Pending |
| `GET/POST /api/ca/staff` | Pending |
| `PUT/DELETE /api/ca/taxpayers/{id}` | Pending |
| Notice logging form in `ca_dashboard.html` | Pending |
| AIS JSON file upload (replace hardcoded sample) | Pending |
| Show `staff_count`, `advance_tax_due_count`, entity breakdown in UI | Pending |
| Staff dropdown from API (replace free text) | Pending |

---

## Phase 3 — Per-Client Intelligence ⬜ NOT STARTED

| Task | Status |
|------|--------|
| Defense brief from taxpayer's actual invoice vendors | Pending |
| Dynamic `audit_readiness_score` | Pending |
| Set `ADVANCE_TAX_DUE` status from installment deadlines | Pending |
| Seed CA demo data on fresh install | Pending |

---

## Phase 4 — Tests & Hardening ⬜ NOT STARTED

| Task | Status |
|------|--------|
| Integration test: AIS JSON file upload end-to-end | Pending |
| Notice API + UI integration test | Pending |
| CI runs `test_ca_firm.py` alongside `test_production.py` | Pending |

---

## Changelog

| Date | Phase | Notes |
|------|-------|-------|
| 2026-07-25 | Phase 1 | Per-taxpayer ledger isolation, scoped AIS, bulk reconcile + advance tax fixes, tests updated |
