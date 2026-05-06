# Double-Entry Ledger design in payments


## Context

Most fintech engineering teams build a payments table first and a ledger later, or may be after the first reconciliation break, or the first support ticket that cannot be answered, or when a regulator asks for an audit trail that does not exist.

Retrofitting double-entry into a system not designed for it is one of the most expensive engineering projects a fintech undertakes. The schema decisions made in week two of an MVP determine whether the system is auditable, reconcilable, and correct three years later.

> This is not an accounting primer. It assumes you know what ***double-entry*** means, and only covers the decisions that are hard, and where teams generally get it wrong.

---

## The decisions that actually matter

Before any schema and code, these are the four decisions that determine the character of your ledger:

**1. Immutability is non-negotiable.** Ledger entries must never be updated or deleted. Corrections are posted as reversals, new entries that cancel the original. If your ORM makes it easy to `UPDATE` a ledger row, you might have a governance problem, not just an engineering one. Regulators do not accept "we corrected it" without an audit trail showing what changed and when.

**2. Balances are derived, not stored.** Computing balances from entries is the correct default. It eliminates an entire class of synchronisation bug. The moment you store a balance alongside the ledger, you have two sources of truth and a window for them to diverge. 

**3. The write path and read path have different requirements.** Posting a transaction needs strong consistency and isolation. Reading a balance or generating a report does not. Coupling both to the same transactional database is the most common ledger architecture mistake in production. CQRS: separating the write model from the read model becomes necessary earlier.

**4. Reconciliation is an engineering pipeline, not a finance team exercise.** If your reconciliation happens monthly, in a spreadsheet, run by someone in finance, it is like accumulating breaks invisibly. **The Synapse Financial collapse** in 2024 was a ledger-reconciliation failure at scale. Automated daily reconciliation with exception alerting is an engineering obligation, not a nice-to-have.

---

## Schema

```sql
-- Accounts: any entity that holds a balance
-- customer wallets, merchant settlement, platform revenue,
-- suspense accounts, fee pools, clearing accounts
CREATE TABLE accounts (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference    VARCHAR(255) NOT NULL UNIQUE,
    account_type VARCHAR(50)  NOT NULL,
    currency     CHAR(3)      NOT NULL,
    status       VARCHAR(20)  NOT NULL DEFAULT 'active',
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    metadata     JSONB
);

-- Transactions: the business event header
-- one transaction produces two or more ledger entries
CREATE TABLE transactions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference        VARCHAR(255)  NOT NULL UNIQUE,  -- idempotency key
    transaction_type VARCHAR(50)   NOT NULL,
    status           VARCHAR(20)   NOT NULL DEFAULT 'posted',
    currency         CHAR(3)       NOT NULL,
    amount           NUMERIC(19,4) NOT NULL CHECK (amount > 0),
    description      TEXT,
    posted_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    metadata         JSONB
);

-- Ledger entries: the immutable accounting record
-- this table is never updated or deleted
CREATE TABLE ledger_entries (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID          NOT NULL REFERENCES transactions(id),
    account_id     UUID          NOT NULL REFERENCES accounts(id),
    entry_type     VARCHAR(6)    NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount         NUMERIC(19,4) NOT NULL CHECK (amount > 0),
    currency       CHAR(3)       NOT NULL,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ledger_entries_account     ON ledger_entries (account_id, created_at DESC);
CREATE INDEX idx_ledger_entries_transaction ON ledger_entries (transaction_id);
```

**Three decisions embedded in this schema worth calling out:**

Amounts are always positive and sign is carried by `entry_type`. This eliminates the class of bug where a negative credit and a positive debit are confused. It also makes the balance calculation unambiguous.

`NUMERIC(19,4)` for monetary values are non-negotiable. Floating point arithmetic is not exact and rounding errors accumulate. Four decimal places covers most currencies; adjust the scale for currencies with unusual precision requirements (JPY uses 0, KWD uses 3).

No need for a `balance` column on `accounts`, read the **"balance caching"** section below.

---

## Enforcing invariant, the write path

The double-entry invariant, every transaction's debits equal its credits, must be enforced at the database level, not just in application code. Application-level invariants fail silently under concurrent load.

```sql
-- Deferred constraint trigger: fires after transaction commit,
-- allows both entries to be inserted before checking balance
CREATE OR REPLACE FUNCTION check_transaction_balance()
RETURNS TRIGGER AS $$
DECLARE
    debit_total  NUMERIC;
    credit_total NUMERIC;
BEGIN
    SELECT
        COALESCE(SUM(amount) FILTER (WHERE entry_type = 'DEBIT'),  0),
        COALESCE(SUM(amount) FILTER (WHERE entry_type = 'CREDIT'), 0)
    INTO debit_total, credit_total
    FROM ledger_entries
    WHERE transaction_id = NEW.transaction_id;

    IF debit_total <> credit_total THEN
        RAISE EXCEPTION 'Unbalanced transaction %: debits=% credits=%',
            NEW.transaction_id, debit_total, credit_total;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER enforce_double_entry
    AFTER INSERT ON ledger_entries
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW EXECUTE FUNCTION check_transaction_balance();
```

> `DEFERRABLE INITIALLY DEFERRED` is the critical detail. Without it, the constraint fires after each individual insert, which means the first entry of a pair always fails the balance check. Deferred execution means it fires at transaction commit, after both entries exist.

```java
@Service
@RequiredArgsConstructor
public class LedgerService {

    private final TransactionRepository transactionRepository;
    private final LedgerEntryRepository ledgerEntryRepository;

    @Transactional
    public Transaction post(
            String reference,
            UUID debitAccountId,
            UUID creditAccountId,
            BigDecimal amount,
            TransactionType type) {

        // Idempotency — return existing if already posted
        return transactionRepository.findByReference(reference)
            .orElseGet(() -> createEntries(
                reference, debitAccountId, creditAccountId, amount, type));
    }

    private Transaction createEntries(
            String reference, UUID debitId, UUID creditId,
            BigDecimal amount, TransactionType type) {

        if (debitId.equals(creditId)) {
            throw new InvalidTransactionException("Debit and credit accounts must differ");
        }

        Transaction tx = transactionRepository.save(Transaction.builder()
            .reference(reference)
            .transactionType(type)
            .amount(amount)
            .postedAt(Instant.now())
            .build());

        ledgerEntryRepository.saveAll(List.of(
            LedgerEntry.builder()
                .transactionId(tx.getId()).accountId(debitId)
                .entryType(EntryType.DEBIT).amount(amount).build(),
            LedgerEntry.builder()
                .transactionId(tx.getId()).accountId(creditId)
                .entryType(EntryType.CREDIT).amount(amount).build()
        ));

        return tx;
    }
}
```

> Retry storms on the write path without idempotency produce duplicate ledger entries that are extremely difficult to unwind.

---

## CQRS, separating write and read

The read-write coupling problem can surface earlier than most teams expect; balance queries and reporting running against the same transactional database as posting operations, or read traffic contending with write traffic. 

The fix is CQRS: the write model (the `ledger_entries` table) is the source of truth. A separate read model, a materialised projection, serves balance queries and reports.

```
Write path:
  POST /payments -> LedgerService -> ledger_entries (PostgreSQL, strong consistency)
                                 -> publishes LedgerEntryPosted event

Read path:
  GET /accounts/{id}/balance -> balance_projections (read replica or separate store)
                             -> updated by consuming LedgerEntryPosted events
```

This does not require a full event sourcing architecture. A simple projection table updated by a Kafka consumer is sufficient for most teams:

```sql
-- Read model: materialised balance per account
-- updated asynchronously from ledger entry events
CREATE TABLE balance_projections (
    account_id     UUID PRIMARY KEY REFERENCES accounts(id),
    balance        NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency       CHAR(3)       NOT NULL,
    last_entry_id  UUID,          -- high-water mark for projection verification
    projected_at   TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
```

The `last_entry_id` high-water mark is essential. It allows you to verify that the projection is current and to detect if the consumer has fallen behind. If the projection drifts from the ledger, you can rebuild it by replaying from the last verified entry.

Two disciplines to maintain:

**The projection is never the source of truth for financial decisions.** Limits, fraud checks, and regulatory reporting always query the ledger or a verified projection. The projection is for read performance, not for authoritative state.

**Monitor the lag.** If the consumer falls behind, balances in the read model become stale. Alert on consumer lag above a threshold, not just on consumer failure is a must.

---

## Balance caching trade-off

If you do not implement CQRS, but your balance queries are becoming a bottleneck, a simpler balance cache can work with one strict requirement.

```sql
CREATE TABLE account_balance_cache (
    account_id    UUID PRIMARY KEY REFERENCES accounts(id),
    balance       NUMERIC(19,4) NOT NULL DEFAULT 0,
    currency      CHAR(3)       NOT NULL,
    last_entry_id UUID,
    updated_at    TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
```

The requirement: **the cache must be updated in the same database transaction as the ledger entries.** Not in a subsequent call, not in a background job, in the same `@Transactional` boundary. If the process crashes between writing ledger entries and writing the cache, the cache is wrong and will be wrong until system restart or hard refresh.

```java
@Transactional
public Transaction post(...) {
    Transaction tx = createEntries(...);          // writes ledger_entries
    updateBalanceCache(debitId, creditId, amount); // same transaction — both commit or neither
    return tx;
}
```

---

## TigerBeetle, if (and when) to consider it

TigerBeetle is a purpose-built financial accounting database. It is not a general-purpose database with ledger logic on top, it implements double-entry primitives natively, enforces the accounting invariant at the storage layer, and is designed for exactly one workload: high-throughput, low-latency financial transaction processing.

**What it does differently:**
- Double-entry is a first-class primitive; accounts, transfers, two-phase commits are built in
- Designed for 10,000+ transfers per second with strict durability guarantees
- Deterministic execution, same inputs always produce the same outputs, which matters for audit and replay
- No general-purpose query language, you work with its transfer and account primitives

**When it makes sense:**
- Your PostgreSQL ledger is genuinely becoming a performance bottleneck at scale
- You are building a new ledger from scratch and want the accounting invariants enforced below the application layer
- Your team has the operational capability to run and maintain a specialised database

**When it does not:**
- You need rich querying, joins, or reporting against the ledger, TigerBeetle is a general-purpose analytics store; you will need a separate layer for that
- Your team is small and operational simplicity matters more than raw performance
- You are retrofitting, TigerBeetle requires a specific data model; migrating an existing ledger to it is non-trivial

The pattern gaining traction in 2025/2026 is a two-layer approach: TigerBeetle as the high-speed transaction layer, PostgreSQL or a data warehouse as the metadata-rich accounting and reporting layer. The two are kept in sync via an event stream. This gives you TigerBeetle's performance and invariant enforcement without sacrificing the queryability you need for reconciliation and compliance.

---

## Multi-currency

Multi-currency adds one hard problem: the exchange rate that applied when money moved is the only rate that matters. A rate looked up at query time for a historical transaction is a reconstruction error.

```sql
-- Extend transactions for FX
ALTER TABLE transactions ADD COLUMN source_currency  CHAR(3);
ALTER TABLE transactions ADD COLUMN source_amount    NUMERIC(19,4);
ALTER TABLE transactions ADD COLUMN target_currency  CHAR(3);
ALTER TABLE transactions ADD COLUMN target_amount    NUMERIC(19,4);
ALTER TABLE transactions ADD COLUMN exchange_rate    NUMERIC(19,10);  -- high precision
ALTER TABLE transactions ADD COLUMN rate_locked_at   TIMESTAMPTZ;
ALTER TABLE transactions ADD COLUMN rate_provider    VARCHAR(100);
```

Three decisions that matter in multi-currency:

**Lock and store the rate at posting time.** The transaction record must contain the exact rate used. This is not just good practice, but required for any dispute resolution, regulatory inquiry, or audit.

**Use clearing accounts for FX legs.** A cross-currency payment between a GBP source account and EUR destination account posts through GBP and EUR clearing accounts. The clearing accounts absorb the conversion. 

**Use `HALF_EVEN` rounding consistently.** `HALF_UP` accumulates rounding bias over large transaction volumes. `HALF_EVEN` (Banker's rounding) distributes it. Make this a team standard, not an individual developer decision, one inconsistent rounding call in a fee calculation compounds into a reconciliation break at scale.

```java
BigDecimal converted = sourceAmount
    .multiply(exchangeRate)
    .setScale(targetCurrencyScale, RoundingMode.HALF_EVEN);
```

---

## What I'd Recommend

**Start with derived balances, not cache.** Add caching only when you have a measured, production performance problem. The synchronisation bug that emerges from premature balance caching is subtle, takes days to detect, and is painful to audit.

**Enforce the invariant at the database level.** The deferred constraint trigger is not optional in a production system. Application-level enforcement fails under concurrent load exactly when you can least afford it.

**Build CQRS into the design early, even if you do not implement the read model immediately.** Design the write path to publish events. When you need the read model (and you will), you can build the projection without changing the write path.

**If you are starting from scratch and expect significant volume, evaluate TigerBeetle seriously.** Not as a replacement for your entire data stack, as a specialised write layer. The engineering investment is real, but so is the operational simplicity of having the accounting invariants enforced below the application layer.

**Treat reconciliation as an engineering team responsibility.** Daily automated reconciliation with exception alerting is not a finance tool, it is a production health signal. If reconciliation breaks are accumulating undetected between monthly reviews, you are operating with incomplete information about the state of your system.

---

## Real-world Failure Mode

Synapse Financial, a US Banking-as-a-Service provider, collapsed in 2024. A significant contributing factor was a ledger-reconciliation failure: the records of who owned what across Synapse's system did not match the actual funds held at partner banks. The discrepancy was not a single event, it was a gap that accumulated over time, invisible between reconciliation cycles, until the mismatch was large enough.

End users, people with real money in accounts, could not access their funds while administrators attempted to reconcile records that had no reliable source of truth to reconcile against.

The engineering lesson is not that ledgers are hard. It is that a ledger without continuous automated reconciliation is not a ledger, it is a record-keeping system with an unknown error rate. The difference between the two only becomes apparent when something goes wrong and you need to know exactly where the money is.

---

## Further Reading

- [TigerBeetle: Debit/Credit Concepts](https://docs.tigerbeetle.com/concepts/debit-credit/)
- [Martin Fowler: Accounting Patterns](https://martinfowler.com/eaaDev/AccountingNarrative.html)
- [Monzo Engineering: How We Built Transaction Isolation](https://monzo.com/blog/2017/11/02/how-we-built-transaction-isolation) 
- [Synapse Financial Collapse: Analysis](https://www.yalejournal.org/publications/the-synapse-collapse)
- [Designing Data-Intensive Applications: Chapter 7 - Transactions](https://dataintensive.net/) 
