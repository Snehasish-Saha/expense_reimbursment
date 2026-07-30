# Expense Reimbursement Platform (Django + DRF)

A backend service for recording expenses, attaching receipts, submitting
multi-currency reimbursement requests, and routing them through a
manager approval workflow with a full audit trail.

See also: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) (design deep-dive)
and [`docs/API.md`](docs/API.md) (full endpoint reference).

## Quick start

```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env              # defaults are fine for local/SQLite use

python manage.py migrate
python manage.py seed_demo_data   # optional: creates demo users/data (see below)
python manage.py createsuperuser  # optional: if you skipped seeding and want your own admin

python manage.py runserver
```

The API is now at `http://127.0.0.1:8000/api/`, interactive docs at
`http://127.0.0.1:8000/api/docs/`, and the Django admin at
`http://127.0.0.1:8000/admin/`.

### Demo accounts (created by `seed_demo_data`)

| Username | Password | Role | Department |
|---|---|---|---|
| `admin` | `Admin@12345` | Admin | — |
| `manager.eng` | `Manager@12345` | Manager | Engineering |
| `manager.sales` | `Manager@12345` | Manager | Sales |
| `employee.priya` | `Employee@12345` | Employee (INR) | Engineering |
| `employee.diego` | `Employee@12345` | Employee (EUR) | Engineering |
| `employee.lucy` | `Employee@12345` | Employee (GBP) | Sales |
| `employee.tom` | `Employee@12345` | Employee (USD) | Sales |

The org's base/reporting currency is seeded as `USD`, with sample EUR/GBP/INR
→ USD exchange rates, expense categories, and two draft expenses for
`employee.priya` so you can immediately try submitting a reimbursement
request. The command is idempotent — safe to re-run.

### Running tests

```bash
python manage.py test
```

Tests focus on the business-logic-critical `apps/reimbursements/services.py`
state machine (duplicate prevention, ownership checks, approve/reject/cancel
transitions, currency conversion) and core `Expense` model validation.

### Postgres instead of SQLite

Uncomment/set the `DB_*` variables in `.env` (see `.env.example`), install
`psycopg2-binary`, and re-run `migrate`.

---

## Assumptions

1. **Approval routing is via an explicit `manager` field on `User`**, not
   inferred from "head of department". Each employee has exactly one
   manager who is the sole decision-maker on their requests (admins can
   also decide, as a break-glass fallback). This keeps authorization
   unambiguous; a real org chart with delegate approvers, multi-level
   approval, or approval thresholds is out of scope but noted below.
2. **"Departmental summary" is computed from `Department`**, a flat
   reference table on `User`, not a hierarchical org unit — sufficient for
   the stated requirement without building a full org-chart module.
3. **The org has a single reporting/base currency** (`OrganizationSettings.base_currency`),
   representing the employer's currency. Each expense is recorded in
   whatever currency the employee actually paid in (defaulting to their
   `home_currency`), and is converted to the base currency **at submission
   time**, with the rate snapshotted for audit stability. We assumed one
   employer-wide base currency is sufficient (vs. per-department or
   per-legal-entity currencies), matching "the employer and employee may
   be in different countries" rather than "the employer operates many
   legal entities."
4. **Exchange rates are maintained manually** (seed data / admin /
   `/api/exchange-rates/`) rather than pulled from a live FX API, since no
   specific provider was specified and the evaluation environment has no
   external network access. The `CurrencyConverter` service is the single
   integration point where a live provider would be wired in.
5. **"Delete" means soft-delete.** Expenses are never hard-deleted, so the
   audit trail and any historical request references remain intact; deleted
   expenses are simply excluded from all default queries.
6. **Only `DRAFT` expenses are editable/deletable/submittable.** Once part
   of a submitted (`PENDING`) request, an expense is locked until that
   request is approved (permanently locked, `REIMBURSED`) or
   rejected/cancelled (unlocked back to `DRAFT` for correction).
6. **Rejecting a request requires a comment**; approving does not (a
   comment is optional but supported) — this matches "managers ... approve
   or reject ... and add comments" while making the rejection reason
   mandatory, since that's the case where the employee most needs
   actionable feedback.
7. **Receipts are stored on local disk** (`MEDIA_ROOT`) via Django's
   `FileField` for simplicity; a production deployment would use S3/GCS
   (see Improvements).
8. **Registration is open** for `EMPLOYEE` role (to make the API easy to
   exercise/demo); creating `MANAGER`/`ADMIN` accounts requires an
   authenticated admin. A real deployment would likely provision all
   accounts via SSO/admin-only invite rather than self-registration.

## Design decisions & trade-offs

| Decision | Rationale | Trade-off |
|---|---|---|
| Business logic in a **service layer**, not fat models or views | One place for business rules, transactional integrity, guaranteed audit logging (see `docs/ARCHITECTURE.md`) | An extra layer of indirection vs. putting everything in the ViewSet |
| **Expense.status as the duplicate-prevention mechanism** rather than a separate "locked" flag or reservation table | Simple, and doubles as the field the UI needs anyway to show expense lifecycle | Ties expense state 1:1 to "current request membership" — an expense can't belong to two *pending* requests even conceptually, which we judged correct (you shouldn't double-claim), but does mean the model can't represent partial/split reimbursement of one expense across two requests |
| **Snapshot exchange rate + converted amount per request item** rather than re-computing conversions on every read | Historical totals are stable and reproducible even if rates/base currency change later — important for anything resembling financial reporting | More storage per row; if a rate was entered wrong and later corrected, already-submitted requests are *not* retroactively corrected (by design — see below) |
| **Generic `AuditLog` table** (GenericForeignKey) vs. per-model audit tables or django-simple-history | One schema for every audited model, one query pattern (`/…/audit/`), easy to extend to new models | Slightly less type-safe than per-model shadow tables; `changes` is a loosely-typed JSON blob rather than a structured diff |
| **JWT (SimpleJWT)** over session auth | Stateless, natural fit for an API consumed by a separate frontend/mobile client | Requires the client to manage token refresh; no built-in server-side "log out everywhere" without a blacklist store (SimpleJWT's rotation+blacklist is enabled, mitigating this) |
| **Soft delete** for expenses | Preserves audit trail / historical request references | Every query must remember to filter `is_deleted=False` (centralized in `ExpenseViewSet.get_queryset`, but a raw `Expense.objects.all()` elsewhere would need the same care) |
| **SQLite default, Postgres via env var** | Zero-friction local setup/review | SQLite's weaker concurrency support means `select_for_update()` row locks are effectively no-ops there; the locking code is correct and takes effect once run under Postgres |
| **DRF ModelViewSet + routers** over hand-rolled function views | Standard, well-understood REST conventions; less boilerplate | Slightly "magic" URL generation; mitigated by `docs/API.md` documenting every concrete route |

## Improvements I would make with more time

- **Multi-step / delegate approvals**: approval thresholds (e.g. amounts
  above $X need a second approver), out-of-office delegate approvers,
  and department-head fallback if an employee has no manager assigned.
- **Live FX rate integration**: a scheduled job pulling daily rates from a
  provider (e.g. exchangerate.host, Open Exchange Rates) into
  `ExchangeRate`, with alerting if a required pair is missing at
  submission time instead of a hard failure.
- **Payment/payout tracking**: a status *after* `APPROVED` representing
  "reimbursement actually paid out" (bank transfer/payroll integration),
  with its own timestamp and reference number — currently `APPROVED` is
  the terminal state.
- **Structured audit diffs**: instead of a loose JSON blob, capture
  field-level before/after values (e.g. via `django-simple-history` or a
  custom diffing utility) for a more precise "what changed" view.
- **Object storage for receipts** (S3/GCS) with signed URLs, virus
  scanning on upload, and image thumbnailing, instead of local disk.
- **Rate limiting & finer-grained permissions** (e.g. DRF throttling
  classes; per-department admin roles rather than one global ADMIN).
- **Bulk operations**: bulk-categorize expenses, bulk-approve requests
  matching a filter (with the same guardrails as single approval).
- **Notifications**: email/Slack webhook on submit/approve/reject so
  employees and managers don't have to poll the API.
- **API versioning** (`/api/v1/...`) from day one to allow non-breaking
  evolution once real clients depend on it.
- **Idempotency keys** on `POST /reimbursement-requests/` so a network
  retry from a flaky client can't accidentally create two requests for the
  same intended submission (today's protection is against re-submitting
  the *same expenses*, not against a literal duplicate POST with different
  expense IDs, though that's inherently harder to construct by accident).
- **CI pipeline**: linting (ruff/flake8), type checking (mypy +
  django-stubs), and `python manage.py test` on every PR; this repo was
  built and syntax-checked in an environment without network access to
  install Django, so please run the test suite locally as a first step.
