# Engineering Practices

## Testing

**306 automated tests, 948 assertions**, run against a real MySQL database, not an in-memory
substitute, because constraint enforcement, locking and SQL behaviour are exactly the things
worth testing. The suite builds the schema **by running every migration from an empty
database**, so a migration that only works on an already-evolved database fails CI.

### What the tests cover

| Area | Tests | Examples of what is asserted |
|---|---:|---|
| **Financial correctness** | 84 | Every non-cash payment path posts to a ledger · wallet accounts settle correctly · ledger direction can't be inverted · per-item tax sums to the header · credit-sale revenue prorates partial payments · finalized records refuse edits · clients can't set prices |
| **Security & access control** | 61 | Middleware order is fixed · segregation of duties across every approval flow · TOTP replay rejected · sessions die on password change · no reset enumeration · stored XSS renders inert · CSP header content · API endpoints require permission · no cross-tenant report leakage |
| **Reporting & analytics** | 56 | Every report type exports correctly through the queue · date-range caps · dashboard figures · forecasting model recovers known relationships |
| **Concurrency & idempotency** | 41 | A double-submitted approval or receipt takes effect once · row locks are proven to execute · document-number generation is locked · branch transfers reserve stock at dispatch · replayed payments post once |
| **Operations & resilience** | 34 | Backup failures raise alerts · off-site copy verified · restore flow is permission-gated · cached settings don't leak between requests |
| **Domain workflows** | 30 | End-to-end POS checkout · procurement lifecycle · stock and store transfers · master-data management |

### Regression discipline

Production defects are fixed together with a regression test that reproduces them, and
the test's docblock records what happened and why. The suite is therefore also a history of the
failure modes the system has already survived, and a guard against any of them returning.

### Load testing

POS checkout and a mixed smoke workload have scripted load tests (k6) that ramp to 50
concurrent users, each with a distinct login. The first run found a real race that the
functional tests had not: concurrent checkouts could compute the same document number. It
was fixed across all four document types that shared the pattern, and the fix was confirmed
under the same load.

## Data integrity by construction

- **Foreign keys are enforced by the database.** Enforced constraints went from 2 to 157
  across the schema's hardening. Each relationship carries a deliberate `ON DELETE` rule:
  `CASCADE` for owned children, `RESTRICT` where deletion would destroy financial history,
  `SET NULL` for optional links.
- **Money is `DECIMAL`**, never floating point.
- **Soft deletes** on master data and financial documents, so history stays reconstructible.
- **Document numbers** are generated under lock and unique per organization.

## Concurrency control

Read-modify-write paths on shared state take row locks inside a transaction: account
balances, stock quantities, approval and receipt transitions, and document-number sequences.
Each guarded action has a test that invokes it twice against the same record. The test asserts
the second call is rejected and the side effect happens exactly once, and it captures the
executed SQL to prove the `FOR UPDATE` lock really ran rather than trusting the code's intent.
True parallel contention is then exercised by the load tests above. See
[FINANCIAL_INTEGRITY.md](FINANCIAL_INTEGRITY.md) for the ledger case.

## Continuous integration

Every push runs the full suite in CI against MySQL 8, from a clean environment:

```
checkout → install dependencies → generate app key → run all 306 tests on MySQL 8
```

Code style follows Laravel Pint.

## Independent review

The system has gone through repeated **independent production-readiness audits**, each
scoring security, correctness, data integrity, operability and performance. Findings are
tracked individually to closure, and every fix is **re-audited**, because a fix can introduce
its own defect. Status is only reported as done once it has been verified against the running
system. A self-reported "done" doesn't count.

## Deployment & operations

- **Zero-surprise deploys.** Front-end assets are built in a controlled environment and shipped
  as artifacts, never compiled on the production host.
- **Scheduled operations** (backups, off-site verification, alerts, reconciliation) run from
  the framework scheduler. A failed backup or a failed off-site check alerts administrators.
- **Backups** are daily, retained 14 days, replicated off-site and verified automatically.
- **Configuration** lives in environment variables. Secrets are kept out of version control,
  and selected sensitive columns are encrypted with a key held outside the database.
