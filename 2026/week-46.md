## 2026-10-25 · Session 4 · Double-entry fundamentals

**Did:** No code. Worked through the accounting model properly after repeatedly
failing to make debit/credit stick.

**Learned:**

### Why double-entry exists
Recording a balance loses information — you overwrite the number and the "where
did it go" is gone, and nothing catches tampering. So instead: record
*movements*, and record both ends. Money never appears or vanishes, it comes
FROM somewhere and goes TO somewhere. If both ends are always written down, then
across the whole ledger everything that left must equal everything that arrived.
Any mismatch is an error you can find.

### Debit and credit are not "in" and "out"
- **Debit** = the destination end (left column) — where value went TO
- **Credit** = the source end (right column) — where value came FROM

That's the whole definition. They're 500-year-old Italian bookkeeping words for
left and right. My confusion came from reading my bank statement, which is
HDFC's ledger — where my account is *their liability*, so all the signs invert.
My intuition wasn't wrong, it was correctly learned from the wrong book.

### Why Assets = Liabilities + Equity is a tautology
- Assets and expenses are where value **lands**
- Liabilities, equity and revenue are where value **originates**

Value doesn't only come from other pots of money. Take a loan → cash arrives,
and the *source* is the fact that you now owe. Earn a fee → cash arrives, and the
source is the fact that you earned it. So "everything you hold" must equal "where
it all came from." Not a formula to memorize.

### Which direction moves a balance
An account's natural side is where value accumulates for that type:
- Assets/expenses live on the debit side → **debits increase them**
- Liabilities/equity/revenue live on the credit side → **credits increase them**

Natural side = where the balance normally rests, NOT the only direction the
account ever receives. An asset gets credited whenever value leaves it.

Repaying a loan is the one that tests this: value flows INTO the debt to
extinguish it, so the liability is *debited* and *decreases*.

### Revenue/liability accounts aren't pots of money
Collecting a ₹20 fee isn't "cash lands in platform_fees". It's:

## 2026-11-14 · Session 5 · ledger-engine

**Did:** Implemented idempotency key handling. Added migration 002
(`request_fingerprint` column), computed a SHA-256 fingerprint of the request,
and handled the 23505 unique violation to return 200 on replay / 409 on
conflict. Verified: three requests, one transaction, two entries.

**Learned:**

### Cache was the wrong instinct
My first thought for storing the payload-to-compare was a cache. Wrong for two
reasons: it's a second source of truth that can disagree with the database (txn
commits, cache write fails → guarantee gone), and idempotency has to be as
durable as the money it protects. The answer was a column on a row I was already
writing.

### A failed query poisons the whole transaction
Postgres: "current transaction is aborted, commands ignored until end of
transaction block." So I couldn't look up the original transaction from inside
the catch — had to ROLLBACK first, then query outside any transaction.

### Concurrent duplicates don't even reach 23505
If two requests with the same key arrive at once, the second one *blocks* at the
INSERT until the first commits or rolls back. First commits → second gets 23505 →
returns the original. First rolls back → second's insert succeeds → processes
normally. The DB serializes it for me. No polling, no "in progress" state to
handle. This is the real argument for insert-and-catch over check-then-insert.

### Adding a NOT NULL column to a live table
Three steps, three deploys, in order:
1. `ALTER TABLE ... ADD COLUMN x TEXT` (nullable — instant, no table rewrite)
2. Backfill existing rows in batches of ~1000 (a single UPDATE over millions of
   rows holds locks and writes huge WAL in one transaction)
3. `ALTER COLUMN x SET NOT NULL`

App code must be writing the column BEFORE the backfill, or new rows arrive NULL
during it. Out-of-order deploys are how teams cause outages.

### Legacy rows can't be fully retrofitted
I can't compute a real fingerprint for transactions created before the column
existed — the original request wasn't captured, and the entries don't tell me
what the client actually sent. You'd backfill a sentinel like 'legacy' and treat
it as "can't verify, don't 409". Schema decisions early on matter more than they
feel like they should.

### What to hash
`{from_account, to_account, amount, currency}` — not `description`. It doesn't
affect the money, so editing it shouldn't trigger a 409. Deciding what goes into
the fingerprint IS the design decision.

**Also:** three typos cost a round each — wrong column name in the INSERT,
`original.row[0]` instead of `rows[0]`, and `requestfingerprint` without the
underscore. That last one was silently always-undefined, which would have made
every replay return 409.

**Stuck on:** Nothing.

**Next session:** Week 4 — concurrency hardening. Serialization failures (40001)
and the retry loop, then scenarios 1–3: idempotency replay ×100 in parallel,
500 concurrent transfers, overdraw race.
