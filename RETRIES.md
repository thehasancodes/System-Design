# Retries

## What Is a Retry?

A **retry** is a technique in system design where an application automatically repeats a failed operation, such as an API call or database query, in the hope that a subsequent attempt will succeed.

**Example:** If a frontend sends a request to a backend service and the first attempt fails because of a brief network interruption, the system can send the request again rather than immediately showing the user an error.

## Why Do We Need Retries?

Retries improve system reliability and fault tolerance by helping applications recover from transient (temporary) failures, such as:

- **Temporary network issues and timeouts:** Recover from brief connection drops or delayed network packets.
- **Server overload and temporary service unavailability:** Give a struggling service a short window to recover.
- **Rate limits and temporary server errors:** Retry responses such as HTTP `429 Too Many Requests` and `503 Service Unavailable`, while following the server's guidance.

Retries are useful only for transient failures. They do not fix permanent errors such as invalid input (`400`), invalid credentials (`401`), or forbidden access (`403`). A not-found response (`404`) is also usually not retryable unless the application has a specific reason to expect the resource to appear.

## How Does a Retry Mechanism Work?

1. **Send the request:** The client executes the operation.
2. **Check the result:** The system determines whether the attempt succeeded or failed.
3. **Return on success:** The successful response is returned to the caller.
4. **Evaluate the retry policy on failure:** The system checks whether the failure is retryable and whether attempts remain.
5. **Wait and retry, or stop:** If another attempt is allowed, the system waits according to its backoff strategy and retries. Otherwise, it returns the error.

## Retry Policy

A **retry policy** defines when and how a failed operation is retried. Key parameters include:

- **Retryable errors:** Which failures qualify for retries, such as selected HTTP `5xx` responses and `429`, versus non-retryable client errors such as `400`, `401`, and `404`.
- **Maximum attempts:** A cap on total attempts, often between two and five, to prevent unbounded retries. Be clear whether the configured number includes the initial attempt.
- **Backoff strategy and delay:** How long to wait between attempts, such as a fixed delay or an increasing delay.
- **Jitter:** Random variation in the delay to prevent many clients from retrying at the same time.
- **Timeouts and deadlines:** Per-attempt timeouts and an overall deadline for the complete operation.
- **Safety and fallback:** Whether it is safe to repeat the operation and what to do when all attempts are exhausted.

## Types of Retry Strategies

- **Immediate retry:** Retries without waiting. This can help with momentary local glitches, but can add load to a struggling service.
- **Fixed delay:** Waits the same amount of time between attempts, such as two seconds. It is simple and predictable, but clients can become synchronized and retry together.
- **Exponential backoff:** Multiplies the delay after each failure:

  `D_n = min(D_max, D_0 × 2^(n - 1))`

  For example, the delays might be one, two, four, and then eight seconds. This gives an overloaded service progressively more time to recover.

- **Exponential backoff with jitter:** Randomizes the exponential delay to spread retries over time. One common full-jitter policy is:

  `D_n ~ U(0, min(D_max, D_0 × 2^(n - 1)))`

  where `U(a, b)` is a uniform random value between `a` and `b`. This is a common default for distributed systems because it helps prevent synchronized retry spikes.

## The Biggest Problems with Retries

- **Amplified server overload:** If a service is already struggling, retries multiply incoming traffic. For example, a client making up to four total attempts can generate as many as four requests for one logical operation. A slowdown can therefore become a larger outage.
- **Duplicate operations and side effects:** A server may complete an operation even if its response is lost in transit. Retrying a non-idempotent operation, such as a payment, can then cause the side effect to happen more than once.
- **Solution (Idempotency):** Enforcing idempotency (e.g., passing unique Idempotency-Key headers) ensures that repeating an operation with the same parameters produces that exact same final state without duplicate execution.

## Retry Budget and Retry Storms

- **Retry storm:** A failure feedback loop in which many clients retry at once, generating a surge of traffic that further overloads an unhealthy service and prevents it from recovering.
- **Retry budget:** A limit on retry traffic, often expressed as a maximum percentage of recent request volume. For example, a service might suppress additional retries once retries exceed 10% of its recent traffic.

To reduce the risk of retry storms, combine exponential backoff with jitter, bounded attempts, total deadlines, respect for `Retry-After`, circuit breakers, and retry budgets. Retries should be limited to operations that are safe to repeat or protected by idempotency.