# ACID Properties in DBMS

ACID is a set of properties that help database transactions remain reliable.

ACID stands for:

- **Atomicity**
- **Consistency**
- **Isolation**
- **Durability**

## 1. Atomicity

A transaction happens completely or not at all.

**Example:** During a bank transfer, money must be deducted from Alice's account and credited to Bob's account. If the transfer fails before completion, the partial changes should be rolled back.

## 2. Consistency

A transaction moves the database from one valid state to another, preserving defined rules and constraints.

**Example:** If a database rule prevents account balances from becoming negative, a transaction must not leave an account with a negative balance.

Consistency depends on the constraints and business rules the application and database enforce.

## 3. Isolation

Concurrent transactions should not interfere in ways that violate the guarantees of the chosen isolation level.

**Example:** Two users try to buy the last item in stock. The database must coordinate their operations so that both do not successfully reserve the same single item.

### Isolation Levels

The SQL standard defines four traditional isolation levels:

1. **Read Uncommitted** — May allow reading changes that another transaction has not committed.
2. **Read Committed** — Prevents dirty reads, but repeated queries can see different committed data.
3. **Repeatable Read** — Prevents dirty reads and non-repeatable reads; exact guarantees vary by database.
4. **Serializable** — Provides behavior equivalent to some serial execution order. Transactions may be aborted and need to be retried.

### Common Concurrency Problems

- **Dirty read:** A transaction reads another transaction's uncommitted changes.
- **Non-repeatable read:** The same row returns different values when read twice in one transaction because another transaction committed an update.
- **Phantom read:** Repeating a query returns a different set of matching rows because another transaction inserted or removed matching data.

**PostgreSQL note:** PostgreSQL treats `READ UNCOMMITTED` as `READ COMMITTED`. PostgreSQL's `REPEATABLE READ` also prevents phantom reads, and `SERIALIZABLE` transactions can fail with serialization errors that applications may need to retry.

### Practical inventory example

To reserve one item only when stock is available, PostgreSQL can perform the check and decrement atomically:

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 101
  AND stock > 0
RETURNING stock;
```

If no row is returned, the item could not be reserved. If creating an order requires other database changes, use an appropriate transaction to keep those changes consistent.

## 4. Durability

Once a transaction commits, its changes should survive crashes and restarts, subject to the database's durability guarantees.

**Example:** After a payment record is successfully committed, restarting the database should not erase that committed record.

## Example: Bank Transfer

Alice has ₹1,000 and Bob has ₹500. Alice transfers ₹200 to Bob.

Before:

| Account | Balance |
|---|---:|
| Alice | ₹1,000 |
| Bob | ₹500 |
| **Total** | **₹1,500** |

After a successful transfer:

| Account | Balance |
|---|---:|
| Alice | ₹800 |
| Bob | ₹700 |
| **Total** | **₹1,500** |

ACID protects the operation as follows:

- **Atomicity:** Both balance updates commit, or neither does.
- **Consistency:** Database rules and constraints remain satisfied.
- **Isolation:** Concurrent transactions are handled according to the chosen isolation guarantees.
- **Durability:** Committed changes survive failures according to the database's durability guarantees.

## SQL Transaction Example

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 200
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 200
WHERE account_id = 2;

COMMIT;
```

- `BEGIN` starts a transaction.
- `COMMIT` commits its changes.
- `ROLLBACK` can undo uncommitted changes.

This is a simplified illustration. Production code should also validate sufficient funds, check that the expected rows were updated, use appropriate constraints and concurrency controls, and handle errors.

## Quick Summary

| Property | Main idea |
|---|---|
| Atomicity | All changes happen, or none do |
| Consistency | Rules and constraints remain satisfied |
| Isolation | Concurrent transactions receive defined guarantees |
| Durability | Committed changes persist |

**Remember:** ACID helps make database transactions reliable, but applications still need correct business logic, constraints, error handling, and appropriate transaction design.
