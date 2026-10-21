# Saga Pattern — Revision Notes

## 1. What is the Saga pattern?

The **Saga pattern** manages a business transaction that spans multiple independent services by breaking it into a sequence of **local transactions**.

If a later step fails, the workflow runs **compensating transactions** to counteract the business effects of earlier successful steps.

Instead of one giant database transaction covering all services, each service commits its own local transaction.

## 2. Example: E-commerce checkout

Suppose a customer orders a laptop for ₹60,000.

### Happy path

1. **Order service:** Create order with status `PENDING`.
2. **Inventory service:** Reserve one laptop.
3. **Payment service:** Capture payment.
4. **Order service:** Mark order `CONFIRMED`.

### Failure path

If payment fails after the laptop is reserved:

1. The workflow detects payment failure.
2. The inventory service releases the reservation.
3. The order is marked `FAILED`.

The release operation is a **compensating transaction**.

## 3. What is a compensating transaction?

A compensating transaction is a new business operation that counteracts an earlier operation.

| Original operation | Possible compensation |
|---|---|
| Reserve inventory | Release the reservation |
| Book a hotel room | Cancel the booking |
| Capture a payment | Issue a refund, if permitted and appropriate |
| Create a shipment | Request shipment cancellation, if still possible |

Compensation is not always a perfect inverse. For example, a shipment may already have left the warehouse, or a refund may take time to complete.

## 4. Saga flow

```text
Create Order
     |
     v
Reserve Inventory
     |
     v
Process Payment
     |
     +---- success ----> Confirm Order
     |
     +---- failure ----> Release Inventory
                              |
                              v
                         Mark Order Failed
```

A Saga should define the normal path, failure conditions, compensation steps, and how to recover if a compensation itself fails.

## 5. Two Saga implementation styles

### A. Choreography

Services communicate by publishing and responding to events. There is no single central component directing every step.

Example:

```text
OrderCreated
    |
    v
Inventory service reserves item
    |
    v
InventoryReserved
    |
    v
Payment service processes payment
    |
    +---- PaymentSucceeded
    |
    +---- PaymentFailed
                 |
                 v
        Inventory service releases item
```

**Advantages**
- Services can remain loosely coupled.
- Works naturally with event-driven architectures.
- No central orchestrator is required.

**Trade-offs**
- The full workflow can be harder to understand as the number of services grows.
- Event tracing and failure diagnosis can become more complex.

### B. Orchestration

A central **orchestrator** tracks the workflow and tells services which action to perform next.

Example:

```text
Orchestrator
   |
   +--> Reserve inventory
   |
   +--> Process payment
            |
            +--> Success: confirm order
            |
            +--> Failure: release inventory
```

**Advantages**
- Workflow logic is visible in one place.
- Complex branching and compensation rules can be easier to manage.

**Trade-offs**
- The orchestrator needs reliable state management.
- It adds a component that must be monitored and recovered.

## 6. Saga vs. traditional transaction

| Aspect | Traditional database transaction | Saga |
|---|---|---|
| Scope | Usually one database | Multiple services |
| Failure handling | Database rollback | Compensation and recovery workflow |
| Atomicity | All-or-nothing within the transaction boundary | No single global atomic commit |
| Intermediate states | Hidden until commit under typical isolation rules | May be visible to other parts of the system |
| Typical use | Short operations within one database | Long-running, multi-service business workflows |

Each Saga step should still use an appropriate local database transaction. Saga coordinates the steps; it does not replace local transactions.

## 7. Common failure scenarios

### Duplicate events
A message may be delivered more than once. A handler should be idempotent so that duplicate delivery does not duplicate the business effect.

### Timeout with an unknown outcome
A payment request may time out even though the payment was captured. The system should check the payment's actual status before blindly retrying or issuing a refund.

### Compensation fails
Inventory release or another compensating action may fail temporarily. The Saga should persist its state and retry or escalate the operation.

### Service crashes
A service can crash after performing a business operation but before recording or publishing the outcome. Durable state and reliable event-publishing patterns help manage this gap.

### Concurrent operations
Multiple customers may try to reserve the last item. Inventory updates need concurrency controls such as atomic conditional updates or suitable database constraints.

## 8. Best practices

- Persist Saga/workflow state.
- Make steps and event handlers idempotent.
- Use unique operation IDs or idempotency keys.
- Set timeouts and retry transient failures with sensible limits.
- Define compensation for each step where compensation is possible.
- Make compensation itself retryable and observable.
- Use reliable event publishing, such as the transactional outbox pattern, when database updates and event publication must stay coordinated.
- Track workflow status, errors, retries, and correlation IDs.
- Reconcile uncertain outcomes rather than assuming a timeout means failure.

## 9. Important limitations

- A Saga does not provide global ACID atomicity across all services.
- Intermediate states can be visible.
- Compensation may not perfectly reverse an action.
- Some actions are difficult or impossible to compensate fully.
- Business rules may need to prevent other actions while a Saga is in progress.

## 10. Saga and banking

A bank transfer illustrates a similar distributed-systems challenge, but actual banking payment rails use defined payment, ledger, clearing, settlement, reversal, and reconciliation mechanisms. Do not assume every bank transfer is implemented as a Saga.

A reversal is a controlled business operation, not necessarily a database rollback.

## 11. Quick revision

Remember:
- Saga = sequence of local transactions.
- Compensation = a new operation that counteracts an earlier effect.
- Choreography = services coordinate through events.
- Orchestration = a central coordinator directs the workflow.
- Idempotency protects against duplicate effects.
- Durable workflow state and recovery are essential.
- Saga provides a recovery strategy, not global atomicity.

## 12. Self-test

1. What is the difference between a local transaction and a Saga?
2. Why is releasing inventory a compensating transaction?
3. How do choreography and orchestration differ?
4. What should the system do if payment times out but may have succeeded?
5. Why must compensation operations also be retry-safe?
