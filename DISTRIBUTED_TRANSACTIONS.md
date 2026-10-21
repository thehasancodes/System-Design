# Distributed Transactions — Revision Notes

## 1. What is a distributed transaction?

A **distributed transaction** is a business operation that involves changes across multiple independent services, databases, or systems.

Each system may own its own database and manage its own local transactions. The challenge is coordinating the overall operation when one part succeeds and another part fails.

### Example: E-commerce checkout

1. Create an order.
2. Reserve inventory.
3. Process payment.
4. Confirm the order.

If payment fails after inventory is reserved, the system needs a defined recovery strategy.

## 2. Local transaction vs. distributed transaction

| Aspect | Local database transaction | Distributed transaction |
|---|---|---|
| Scope | Usually one database | Multiple systems or services |
| Coordination | Managed by one database | Requires cross-system coordination |
| Failure handling | Database can roll back its transaction | Requires a distributed protocol or workflow |
| Complexity | Usually lower | Higher because of network failures, timeouts, and independent state |

A local SQL transaction cannot normally make changes inside another service's independent database atomic automatically.

## 3. Important challenges

- **Partial failure:** One service succeeds while another fails.
- **Network delays and timeouts:** A response may be delayed or lost even when an operation succeeded.
- **Duplicate requests:** Retries can accidentally repeat an operation.
- **Consistency:** Different services may temporarily have different views of the business operation.
- **Recovery:** The system must know whether to retry, compensate, reject, or reconcile.

## 4. Two common approaches

### A. Two-Phase Commit (2PC)

2PC coordinates participants through two phases.

**Phase 1 — Prepare**
- A coordinator asks each participant whether it can commit.
- Each participant prepares its transaction and votes yes or no.

**Phase 2 — Commit or abort**
- If all participants vote yes, the coordinator instructs them to commit.
- If any participant votes no, the coordinator instructs them to abort.

**Advantages**
- Provides a coordinated all-or-nothing decision when the protocol completes correctly.
- Useful when participants support the protocol and the required consistency model.

**Limitations**
- Participants may remain blocked while waiting for the coordinator or a decision.
- Coordination adds latency and operational complexity.
- It is not always suitable for loosely coupled microservices.

### B. Saga pattern

A Saga divides a long-running business transaction into a sequence of **local transactions**. If a later step fails, the workflow runs **compensating transactions** to counteract earlier successful steps.

Example:
1. Create order.
2. Reserve inventory.
3. Attempt payment.
4. If payment fails, release inventory and mark the order as failed.

A compensation is a new business operation, not a magical rollback across databases.

## 5. 2PC vs. Saga

| Aspect | 2PC | Saga |
|---|---|---|
| Main idea | Coordinate commit/abort across participants | Sequence local transactions and compensate on failure |
| Intermediate state | Participants may wait for a global decision | Intermediate business states can be visible |
| Recovery | Protocol recovery and coordinator decisions | Retries, compensation, and workflow recovery |
| Typical trade-off | Stronger coordinated atomicity, possible blocking | Better fit for long-running workflows, but not globally atomic |
| Common use | Systems where participants support coordinated transactions | Microservices and business workflows |

## 6. Idempotency and retries

A service may time out after completing an operation. The caller might retry without knowing whether the first attempt succeeded.

An **idempotent operation** can be repeated without repeating its intended effect.

Common safeguards:
- Use a stable operation or idempotency key.
- Store operation outcomes durably.
- Use unique constraints to prevent duplicate business records.
- Make state transitions safe to repeat.
- Distinguish a confirmed failure from an unknown outcome.

## 7. Banking example

A transfer between two banks is a useful example of a distributed-systems problem because each bank controls its own systems.

A simplified payment flow may involve:
1. The sender's bank validates and records the outgoing payment.
2. A payment rail routes the instruction.
3. The recipient's bank processes the incoming payment.
4. The payment system records status and follows its rules for failures, reversals, settlement, and reconciliation.

**Important:** Real payment rails do not necessarily use 2PC or Saga directly. They have specific payment, clearing, settlement, ledger, and recovery rules. Clearing, settlement, and crediting a recipient's account are related but distinct operations.

## 8. Key terms

- **Coordinator:** Component that coordinates participants in a protocol such as 2PC.
- **Participant:** System involved in the coordinated transaction.
- **Local transaction:** Atomic operation within one service's database boundary.
- **Compensating transaction:** Business operation that counteracts an earlier successful step.
- **Eventual consistency:** Systems may temporarily disagree but are expected to converge if processing and recovery succeed.
- **Reconciliation:** Comparing records across systems to identify and resolve discrepancies.
- **Idempotency:** Repeating an operation does not repeat its intended effect.

## 9. Quick revision

Remember:
- Distributed transactions span independent systems.
- Partial failure is a central challenge.
- 2PC coordinates a commit-or-abort decision.
- Saga coordinates local transactions and compensation.
- Retries need idempotency and durable state.
- A bank transfer illustrates the general problem, but actual banking systems use payment-rail-specific mechanisms.

## 10. Self-test

1. Why can't a normal SQL transaction automatically update two independent service databases atomically?
2. What happens in the prepare phase of 2PC?
3. How is a Saga different from a traditional rollback?
4. Why can a timeout make payment status uncertain?
5. Why are idempotency and reconciliation important in distributed systems?
