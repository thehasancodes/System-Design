# Idempotency

## 1. What Is Idempotency?

**Idempotency** is a property of an operation where performing it
multiple times has the same final effect as performing it once.

In simple terms:

> You can safely repeat an idempotent operation without changing the
> final result after the first successful execution.

![1792569229053](image/IDEMPOTENCY/1792569229053.webp)

### Everyday example

Imagine a light switch controlled by two commands:

- `TURN_ON`: sets the light to ON.
- `TOGGLE`: changes the light from ON to OFF or OFF to ON.

Repeating `TURN_ON` keeps the light ON, so it is idempotent.

Repeating `TOGGLE` keeps changing the light's state, so it is not
idempotent.

## 2. Idempotent vs. Non-Idempotent Operations

---

Operation Example Idempotent? Why?

---

Set a user's role `role = "admin"` Yes Repeating it
to `admin` leaves the role
as `admin`.

Set a light to ON `light = ON` Yes The final state
remains ON.

Increment a `counter++` No Every execution
counter increases the
counter.

Add \$10 to a `balance += 10` No Every execution
balance adds another
\$10.

Set a balance to `balance = 100` Yes Repeating it
\$100 leaves the
balance at \$100.

Create a new `createOrder()` Usually no Repeating it may
order create multiple
orders.

---

The important point is that **idempotency is about the final effect, not
necessarily whether every response is identical**.

## 3. Why Does Idempotency Matter?

Network requests can fail or time out even when the server has
successfully completed the operation.

For example:

1. A customer clicks **Pay**.
2. The application sends a payment request.
3. The server processes the payment.
4. The response gets lost because of a network problem.
5. The application cannot tell whether the payment succeeded.
6. The application retries the request.

Without protection, the retry might charge the customer twice.

Idempotency helps systems handle retries safely, especially when dealing
with payments, order creation, account updates, and other operations
that must not happen more than intended.

## 4. Idempotency in HTTP Methods

HTTP methods have conventional expectations about idempotency.

---

HTTP method Generally idempotent? Explanation

---

`GET` Yes Retrieves a resource
and should not change
its state.

`PUT` Yes Replaces a resource
with the specified
representation.
Repeating the same
request should leave
the same state.

`DELETE` Yes The resource should
remain deleted. A later
request may return a
different status, but
the intended final
state is the same.

`POST` Usually no Often creates a new
resource or triggers an
action each time.

`PATCH` Depends Some partial updates
are idempotent; others
are not.

---

**Example:** setting a profile's display name to `"Mehedi"` using the
same `PUT` request is generally idempotent. Sending a request that adds
another item to a shopping cart may not be.

These are general conventions. The actual behavior depends on how the
API is designed and implemented.

## 5. Idempotency Keys

An **idempotency key** is a unique value sent with a request so that the
server can recognize retries of the same logical operation.

For example, a client sends:

```http
POST /orders
Idempotency-Key: checkout-001
Content-Type: application/json

{
  "item": "Keyboard",
  "quantity": 1
}
```

If the client retries the same request with the same key, the server can
return the original result rather than creating another order.

A new checkout should use a new key, such as `checkout-002`.

### Typical server-side flow

1. The client generates an idempotency key for a logical operation.
2. The server checks whether that key has already been processed.
3. If it has, the server returns the stored result.
4. If it has not, the server processes the request and stores the
   result associated with the key.
5. A retry with the same key reuses the original result.

A production implementation should also consider how to store keys
durably, handle simultaneous requests, define key expiration, and decide
what happens if the same key is reused with a different request body.

## 6. Practical Node.js Demonstration

This small HTTP server demonstrates how an idempotency key can prevent a
repeated order request from creating a second order.

Save the following code as `idempotency-demo.js`:

```javascript
const http = require("http");

const processedRequests = new Map();
let nextOrderId = 1;

const server = http.createServer((req, res) => {
  if (req.method !== "POST" || req.url !== "/orders") {
    res.writeHead(404, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ error: "Not found" }));
  }

  const idempotencyKey = req.headers["idempotency-key"];

  if (!idempotencyKey) {
    res.writeHead(400, { "Content-Type": "application/json" });
    return res.end(
      JSON.stringify({
        error: "Idempotency-Key header is required",
      }),
    );
  }

  // If this key was already processed, return its original result.
  if (processedRequests.has(idempotencyKey)) {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(
      JSON.stringify({
        message: "Returning the original result",
        order: processedRequests.get(idempotencyKey),
      }),
    );
  }

  // Create and remember one order for this key.
  const order = {
    id: nextOrderId++,
    status: "created",
  };

  processedRequests.set(idempotencyKey, order);

  res.writeHead(201, { "Content-Type": "application/json" });
  res.end(
    JSON.stringify({
      message: "Order created",
      order,
    }),
  );
});

server.listen(3000, "127.0.0.1", () => {
  console.log("Idempotency demo running at http://localhost:3000");
});
```

### Run the server

```powershell
node idempotency-demo.js
```

Keep the terminal running. Open another PowerShell terminal and send a
request:

```powershell
curl.exe -i -X POST http://localhost:3000/orders -H "Idempotency-Key: checkout-001"
```

Run the same command again. The server should return the same order ID
instead of creating a new order.

Now try a different key:

```powershell
curl.exe -i -X POST http://localhost:3000/orders -H "Idempotency-Key: checkout-002"
```

The new key represents a different logical operation, so the demo
creates a new order.

### Limitations of this demo

This is a learning example, not production-ready payment or
order-processing code:

- The `Map` exists only in memory, so restarting the server loses the
  stored keys.
- It does not read or validate a request body.
- It does not handle concurrent requests safely.
- It does not compare request payloads when a key is reused.
- It does not implement key expiration or durable storage.

A real service would typically persist idempotency records in a database
or another shared store and enforce safe processing when requests arrive
at the same time.

## 7. Common Misunderstandings

- **Idempotent does not mean "runs only once."** The operation may
  execute repeatedly; the intended final effect remains the same.
- **Idempotent does not mean the response must always be identical.**
  For example, the first `DELETE` may return `200`, while a later one
  returns `404`; the resource remains deleted.
- **Retries are not automatically safe.** A retry is safe only when
  the operation's semantics and implementation support it.
- **An idempotency key is not magic by itself.** The server must store
  and check keys correctly, including under concurrent requests.

## 8. Quick Knowledge Check

For each scenario, decide whether the operation is idempotent.

1. Setting `volume = 50` every time.
2. Running `likes = likes + 1` every time.
3. Sending the same `PUT` request to set a user's email to
   `user@example.com`.
4. Retrying an order-creation request without an idempotency key.
5. Deleting the same resource multiple times.

### Answers

1. **Idempotent** --- the volume remains 50.
2. **Not idempotent** --- the likes count keeps increasing.
3. **Generally idempotent** --- the same resource state is requested
   each time.
4. **Usually not idempotent** --- multiple orders may be created.
5. **Idempotent in its intended final effect** --- the resource remains
   deleted, even if responses differ.

## Key Takeaway

**Idempotency means that repeating an operation has the same intended
final effect as performing it once.** It is especially useful for
designing reliable APIs and handling retries when network failures make
the result of a request uncertain.
