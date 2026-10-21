# Rate Limiting --- Complete Learning Notes

## Project Learn CSE in Depth

> **Learning approach:** Start from the problem rate limiting solves,
> then study each algorithm, its behavior, trade-offs, and production
> implications. These notes consolidate the rate-limiting lessons
> covered so far. The five-batch plan is included at the end; Batches 1
> and 2 have been studied in detail so far.

------------------------------------------------------------------------

## Table of Contents

1.  [Why Rate Limiting Exists](#1-why-rate-limiting-exists)
2.  [Rate, Count, Burst, and
    Concurrency](#2-rate-count-burst-and-concurrency)
3.  [What Makes a Rate-Limiting
    Policy](#3-what-makes-a-rate-limiting-policy)
4.  [Rate Limiting vs Authentication and
    Authorization](#4-rate-limiting-vs-authentication-and-authorization)
5.  [Fixed Window Counter](#5-fixed-window-counter)
6.  [Sliding Window Log](#6-sliding-window-log)
7.  [Sliding Window Counter](#7-sliding-window-counter)
8.  [Token Bucket](#8-token-bucket)
9.  [Leaky Bucket](#9-leaky-bucket)
10. [Algorithm Comparison](#10-algorithm-comparison)
11. [Atomicity and Race Conditions](#11-atomicity-and-race-conditions)
12. [Practical Policy Design](#12-practical-policy-design)
13. [Express and the Current Backend
    Stack](#13-express-and-the-current-backend-stack)
14. [Limitations and Security
    Warnings](#14-limitations-and-security-warnings)
15. [Knowledge Check](#15-knowledge-check)
16. [Five-Batch Learning Roadmap](#16-five-batch-learning-roadmap)
17. [References](#17-references)

------------------------------------------------------------------------

## 1. Why Rate Limiting Exists

An API is a service that consumes finite resources. Each request can use
CPU time, memory, database connections, network bandwidth, third-party
API quota, or other capacity.

If one client sends requests excessively, it may: - Consume a
disproportionate share of server resources. - Slow down other users. -
Trigger unnecessary database work. - Abuse expensive endpoints. -
Exhaust a quota or create unexpected costs. - Make an application easier
to disrupt.

**Rate limiting** controls how frequently a client or other defined
identity may perform an action over a specified interval or under a
specified traffic policy.

It is a traffic-control mechanism, not a complete security system and
not a guarantee against all denial-of-service attacks.

### Example

Suppose a login endpoint permits a limited number of attempts within a
time interval. Once the configured limit is exceeded, the server
temporarily rejects further attempts according to the policy. The exact
limit and response behavior must be chosen for the application rather
than copied blindly.

------------------------------------------------------------------------

## 2. Rate, Count, Burst, and Concurrency

These ideas are related but not interchangeable.

### Request count

The number of requests observed within a period.

Example: 30 requests were received during one minute.

### Request rate

How frequently requests arrive, often expressed as requests per second
or requests per minute.

A count is meaningful only with its measurement interval. Thirty
requests in one second is very different from thirty requests in one
hour.

### Burst traffic

A burst is a short, concentrated wave of requests. A client may have a
modest long-term average rate but send many requests in a very short
interval.

Some algorithms permit bursts; others smooth or reject them more
aggressively.

### Concurrency

Concurrency is the number of operations currently in progress.

A service can receive requests at a reasonable rate but still accumulate
many simultaneous slow requests. Rate limiting controls arrival
frequency or permitted traffic over time; a concurrency limit controls
how many operations may be in flight at once.

**Key distinction:** Rate limiting and concurrency limiting solve
related but different problems. A robust service may need both.

------------------------------------------------------------------------

## 3. What Makes a Rate-Limiting Policy

A policy needs more than a number. Important design choices include:

### Identity: who is being limited?

Possible keys include: - IP address. - Authenticated user ID. - API key
or client ID. - Account plus endpoint. - A combination of dimensions.

Each choice has trade-offs. IP addresses can be shared by many
legitimate users, and a client may appear under different IPs over time.
Authenticated user IDs are more precise for account-specific policies,
but unauthenticated endpoints need a different strategy.

### Scope: what is being limited?

Possible scopes include: - Entire API. - One route or endpoint. - A
class of expensive operations. - One user or API client. - A combination
such as one user on one endpoint.

A global limit can protect overall capacity, while endpoint-specific
limits can account for different costs. A lightweight read endpoint and
an expensive report-generation endpoint may need different policies.

### Policy dimensions

A policy commonly defines: - **Limit:** How much traffic is permitted. -
**Window or timing behavior:** The interval or algorithm used to measure
requests. - **Identity key:** Which requests count together. -
**Scope:** Which routes or actions are covered. - **Excess behavior:**
What happens after the limit is reached. - **State storage:** Where
counters or token state live. - **Reset/refill behavior:** How capacity
becomes available again.

### Example policy statement

"Allow up to a configured number of requests per authenticated user to
this endpoint during the configured interval, and reject excess requests
with a rate-limit response."

This statement still needs concrete values and implementation decisions,
but it identifies the policy's essential parts.

------------------------------------------------------------------------

## 4. Rate Limiting vs Authentication and Authorization

These controls answer different questions:

  -----------------------------------------------------------------------
  Control                             Question
  ----------------------------------- -----------------------------------
  Authentication                      Who or what is making the request?

  Authorization                       Is that identity allowed to perform
                                      this action?

  Rate limiting                       How frequently may this identity or
                                      traffic key perform it?

  Concurrency limiting                How many operations may be in
                                      progress at once?
  -----------------------------------------------------------------------

A correctly authenticated user can still send too many requests. A
rate-limited request may still be unauthorized. These controls
complement one another.

Rate limiting should not be treated as a replacement for authentication,
authorization, input validation, secure database access, or
infrastructure-level DDoS protection.

------------------------------------------------------------------------

## 5. Fixed Window Counter

### Mechanism

Divide time into fixed intervals, such as one-minute windows. Maintain a
counter for each identity and window.

For a limit of 100 requests per minute: 1. Identify the current time
window. 2. Retrieve the counter for that identity and window. 3. If the
counter is below the limit, increment it and allow the request. 4.
Otherwise, reject the request until a later window.

A key might conceptually contain an identity and window identifier, such
as `user-123:minute-2030`.

### Advantages

-   Simple to implement.
-   Low memory usage.
-   Efficient counter operations.
-   Easy to reason about for basic policies.

### Main weakness: boundary bursts

Suppose the limit is 100 requests per minute: - A client sends 100
requests just before one window ends. - It sends another 100 just after
the next window begins.

The client may send 200 requests in a very short time while remaining
within each window's counter limit.

This is the fixed-window boundary problem. Fixed windows enforce their
per-window counts, but do not strictly smooth traffic across adjacent
boundaries.

### Best fit

A simple, inexpensive policy where this boundary behavior is acceptable.

------------------------------------------------------------------------

## 6. Sliding Window Log

### Mechanism

Store the timestamps of recent requests for each identity. For each new
request: 1. Remove timestamps that fall outside the rolling time
interval. 2. Count the remaining timestamps. 3. If the count is below
the limit, record the current request and allow it. 4. Otherwise, reject
it.

For a limit of 100 requests in any rolling 60-second interval, the
algorithm examines requests within the most recent 60 seconds rather
than aligning to a wall-clock minute.

### Advantages

-   Accurate rolling-window behavior.
-   Avoids the fixed-window boundary burst.
-   Easy to explain conceptually.

### Disadvantages

-   Stores many timestamps instead of one counter per window.
-   Memory and processing costs can grow with request volume.
-   Concurrent requests must be handled atomically to prevent
    overshooting the limit.

### Best fit

Situations that require accurate rolling-window counts and can afford
the associated state and processing cost.

------------------------------------------------------------------------

## 7. Sliding Window Counter

A sliding window counter approximates a rolling window by combining the
current fixed-window count with the previous window's count.

Let: - `C` = count in the current window. - `P` = count in the previous
window. - `f` = fraction of the current window that has elapsed, from 0
to 1.

A common estimate is:

`estimated_count = C + P × (1 − f)`

### Why this formula?

At the start of the current window, most of the previous window is still
inside the rolling interval, so its count receives a high weight. As
time advances, less of the previous window overlaps the rolling
interval, so its contribution decreases.

### Example

Suppose: - Previous-window count `P = 80`. - Current-window count
`C = 30`. - Half of the current window has elapsed, so `f = 0.5`.

Then:

`estimated_count = 30 + 80 × (1 − 0.5) = 70`

The estimated rolling-window count is 70.

This is an approximation. It assumes requests in the previous window are
distributed sufficiently evenly for the weighting to be useful; it does
not retain the exact timestamp of every request.

### Advantages

-   Uses less state than a full timestamp log.
-   Reduces the sharp boundary behavior of fixed windows.
-   Often a practical balance between accuracy and cost.

### Limitations

-   Approximate rather than exact.
-   The estimate can differ from the actual rolling count when requests
    are clustered unevenly.

------------------------------------------------------------------------

## 8. Token Bucket

Token Bucket is a common algorithm for allowing controlled bursts while
constraining the long-term average rate.

### Components

-   **Bucket capacity (`B`):** The maximum number of tokens the bucket
    can hold.
-   **Refill rate (`r`):** How quickly tokens are added.
-   **Current token count (`T`):** The tokens available now.
-   **Request cost:** The number of tokens required for a request, often
    one.

### Mechanism

1.  Tokens accumulate at the configured refill rate, up to the bucket's
    capacity.
2.  An incoming request checks whether enough tokens are available.
3.  If enough tokens exist, the request consumes its token(s) and is
    allowed.
4.  If not, the request is rejected or delayed, depending on the
    implementation.

A bucket with capacity 20 can allow a burst of up to 20 one-token
requests when full. The refill rate controls how quickly capacity
becomes available again.

### Advantages

-   Allows intentional bursts.
-   Controls sustained average traffic.
-   Efficient state: usually a token count and a time reference per
    identity.
-   Useful for API quotas and many network traffic policies.

### Limitations

-   Burst capacity must be selected carefully.
-   Refill and consumption updates must be correct under concurrency.
-   A distributed implementation must coordinate shared state.

### Important distinction

Token Bucket is not the same as a fixed-window counter. A token bucket
models available capacity that replenishes over time, rather than simply
resetting a count at a fixed boundary.

------------------------------------------------------------------------

## 9. Leaky Bucket

"Leaky Bucket" describes related approaches that regulate incoming
traffic using a bucket-like model. Two behaviors are worth
distinguishing.

### Queueing / traffic shaping

Requests or units of work enter a queue and leave at a controlled rate.
Excess arrivals may wait in the queue; if the queue fills, new arrivals
may be rejected.

This smooths bursts into a more regular output stream, but introduces
queueing delay and requires a queue capacity policy.

### Policing / rejection

A related bucket model may reject traffic that exceeds the allowed rate
instead of queueing it.

The exact behavior depends on the implementation and terminology being
used. Do not assume every algorithm called "leaky bucket" has identical
queueing semantics.

### Contrast with Token Bucket

-   **Token Bucket:** Can permit a burst when tokens have accumulated,
    while constraining the longer-term average.
-   **Leaky Bucket shaping:** Emphasizes a controlled output rate;
    bursts may wait in a queue rather than immediately passing through.

### Trade-offs

Shaping can smooth traffic but may increase latency. Policing can reject
excess traffic quickly but does not make excess work wait for later
processing.

------------------------------------------------------------------------

## 10. Algorithm Comparison

  -----------------------------------------------------------------------
  Algorithm         State commonly    Burst behavior    Accuracy / main
                    maintained                          trade-off
  ----------------- ----------------- ----------------- -----------------
  Fixed Window      Counter per       Can burst across  Simple and cheap;
  Counter           identity/window   window boundaries boundary weakness

  Sliding Window    Recent request    Strictly enforces Accurate, but
  Log               timestamps        a rolling count   state can be
                                      when implemented  expensive
                                      atomically        

  Sliding Window    Current and       Smooths boundary  Approximation
  Counter           previous counts   behavior          with lower state
                    plus window                         cost
                    position                            

  Token Bucket      Token count and   Intentionally     Controls
                    refill-time state permits bursts up sustained rate
                                      to capacity       and burst size

  Leaky Bucket      Queue state and   Buffers/smooths   Can add delay;
  shaping           service rate      bursts until      full queue needs
                                      capacity is       a policy
                                      reached           
  -----------------------------------------------------------------------

### How to choose

-   Choose **Fixed Window** when simplicity and low state cost are more
    important than boundary smoothness.
-   Choose **Sliding Window Log** when exact rolling-window counting is
    important and the state cost is acceptable.
-   Choose **Sliding Window Counter** for a lower-cost approximation to
    rolling-window behavior.
-   Choose **Token Bucket** when controlled bursts plus a sustained-rate
    constraint are desired.
-   Choose **Leaky Bucket shaping** when smoothing output traffic is
    important and queueing is acceptable.

No algorithm is universally best. The desired traffic behavior and
operational cost determine the choice.

------------------------------------------------------------------------

## 11. Atomicity and Race Conditions

An algorithm can be logically correct but incorrectly implemented under
concurrent requests.

### The race condition

Imagine the limit is 10 and the stored count is 9. Two requests arrive
at almost the same time.

If both requests: 1. Read the count as 9. 2. Independently decide that
the request is allowed. 3. Increment the count afterward.

Both may be allowed, producing 11 total requests. The intended limit has
been exceeded.

This is a **race condition**: the result depends on the interleaving of
concurrent operations.

### Atomic operation

The check-and-update operation must be performed as one indivisible
operation, or coordinated so concurrent requests cannot make the
decision using the same outdated state.

For distributed rate limiting, storing a counter in Redis does not by
itself guarantee atomicity. The implementation must use an appropriate
atomic command or transaction/script strategy for the chosen algorithm.

### Why process-local state is insufficient for a distributed limit

If each Express instance maintains its own counter, a client can be
counted separately on each instance. A supposed global limit may
therefore be multiplied across instances.

A shared state store plus atomic updates is a common design for
enforcing a limit across multiple backend instances.

------------------------------------------------------------------------

## 12. Practical Policy Design

Before choosing an algorithm, define the desired behavior.

### Questions to answer

1.  **Who is being limited?** IP, authenticated user, API key, or a
    combination?
2.  **What is the scope?** Whole API, specific route, or a class of
    expensive operations?
3.  **What is the limit?** The actual number must follow the endpoint's
    risk and capacity.
4.  **What time behavior is required?** Fixed interval, rolling window,
    refill rate, or controlled output rate?
5.  **Are bursts acceptable?** If yes, how large?
6.  **What happens when the limit is reached?** Reject, delay, queue, or
    return a retry hint?
7.  **Where is state stored?** In-process or shared store?
8.  **What happens across instances?** Must the limit be global or
    instance-local?
9.  **How is the client identified behind proxies?** Proxy configuration
    must be deliberate.
10. **How will the policy be observed?** Track allowed and rejected
    requests, latency, and unusual traffic patterns without logging
    unnecessary sensitive information.

### Endpoint-specific policies

Different endpoints can have different costs and risks: - A simple
public read endpoint may tolerate a relatively generous limit. - Login
or verification endpoints may need abuse-focused controls. - Expensive
report generation may need stricter rate and concurrency controls. -
Upload endpoints may need request-rate limits plus size and processing
constraints.

Actual values should be chosen based on expected legitimate behavior,
capacity, security requirements, and testing. Avoid arbitrary limits
that block normal users.

------------------------------------------------------------------------

## 13. Express and the Current Backend Stack

The current backend direction is JavaScript + Node.js + Express +
PostgreSQL + Prisma, with Redis considered for a later phase.

### Typical Express approach

The `express-rate-limit` package can apply a request limit to the whole
application or selected routes. A typical configuration defines a
window, maximum request count, and response behavior. Excess requests
commonly receive HTTP `429 Too Many Requests`.

The exact configuration should be selected from the package's current
documentation and the application's requirements.

Official project documentation: https://express-rate-limit.mintlify.app/

### Placement and scope

-   Apply policies to the routes that need them.
-   Consider separate policies for login, expensive operations, and
    general API traffic.
-   Do not assume one global limit is suitable for every endpoint.
-   Ensure the identity key matches the intended policy.

### IP addresses and proxies

Behind a reverse proxy, the application may observe the proxy's IP
rather than the original client IP. Proxy trust settings must be
configured carefully. Blindly trusting arbitrary forwarded-IP headers
can let clients manipulate their apparent identity and bypass a limiter.

Shared IP addresses can also represent many legitimate users, so IP-only
limits can cause collateral blocking. Authenticated user or API-client
limits may complement IP-based protection.

### Multi-instance deployments

A default in-memory limiter store can maintain separate counters in
separate processes. If the intended policy must apply across multiple
instances, use a compatible shared store and ensure updates are
coordinated correctly. Redis is one possible store, not an automatic
requirement for every deployment.

------------------------------------------------------------------------

## 14. Limitations and Security Warnings

-   **Not a full DDoS defense:** Application rate limiting cannot
    replace suitable infrastructure and network-level protection.
-   **Identity can be imperfect:** IP addresses may be shared, rotated,
    spoofed through misconfigured proxy handling, or observed
    differently behind gateways.
-   **Per-instance counters can multiply limits:** Shared state may be
    needed for a global policy.
-   **Concurrency matters:** Non-atomic check-and-increment logic can
    overshoot.
-   **Different routes have different costs:** A single global policy
    may be too strict for one route and too loose for another.
-   **Rate limits can harm legitimate users:** Test shared-network
    cases, retries, and realistic bursts.
-   **Rate limiting is not authorization:** A request below the limit
    can still be forbidden.
-   **Queueing has costs:** Shaping excess traffic can increase latency
    and consume memory; queues need capacity limits.
-   **Response behavior should be deliberate:** HTTP 429 is the standard
    response status for too many requests; retry guidance should reflect
    the actual policy.
-   **Observability matters:** Measure rejected traffic and system
    behavior to tune the policy rather than guessing.

------------------------------------------------------------------------

## 15. Knowledge Check

1.  **A client sends 100 requests at the end of a fixed window and
    another 100 immediately after the next window begins. What weakness
    does this show?**\
    **Answer:** The fixed-window boundary burst.

2.  **Which algorithm stores recent request timestamps to enforce a
    rolling-window count?**\
    **Answer:** Sliding Window Log.

3.  **Which algorithm can allow bursts up to a bucket capacity while
    controlling sustained average traffic?**\
    **Answer:** Token Bucket.

4.  **What is the difference between rate limiting and concurrency
    limiting?**\
    **Answer:** Rate limiting controls traffic frequency or volume over
    time; concurrency limiting controls how many operations are in
    progress simultaneously.

5.  **Why can two simultaneous requests exceed a counter limit even when
    the code checks the limit first?**\
    **Answer:** A race condition can allow both requests to read the
    same old count before either update is recorded. The check and
    update need atomic coordination.

6.  **Does using Redis automatically make a distributed rate limiter
    correct?**\
    **Answer:** No. The algorithm's state updates must still be
    coordinated atomically, and the identity and policy must be designed
    correctly.

7.  **Is rate limiting a replacement for authentication or DDoS
    protection?**\
    **Answer:** No. It is one layer of traffic control and should
    complement other controls.

------------------------------------------------------------------------

## 16. Five-Batch Learning Roadmap

The topic is intentionally split into exactly five batches.

  -----------------------------------------------------------------------
  Batch                   Topic                   Status
  ----------------------- ----------------------- -----------------------
  1                       Foundations and Traffic **Covered**
                          Behavior: rate vs       
                          count, bursts,          
                          concurrency, identity,  
                          scope, policy,          
                          security-control        
                          distinctions            

  2                       Algorithms and Internal **Covered**
                          Mechanics: fixed        
                          window, sliding         
                          log/counter, token      
                          bucket, leaky bucket,   
                          race conditions         

  3                       HTTP and Express        Planned
                          Implementation:         
                          middleware placement,   
                          endpoint policies,      
                          status 429, response    
                          headers, package        
                          configuration           

  4                       Redis and Distributed   Planned
                          Rate Limiting: shared   
                          state, atomic updates,  
                          multi-instance          
                          deployments,            
                          consistency             

  5                       Security and Production Planned
                          Design: identity        
                          strategy, proxies,      
                          abuse cases,            
                          observability, testing, 
                          tuning, and design      
                          trade-offs              
  -----------------------------------------------------------------------

The future batches are a roadmap, not claims that those lessons have
already been completed.

------------------------------------------------------------------------

## 17. References

1.  **RFC 6585 --- Additional HTTP Status Codes**\
    Defines HTTP `429 Too Many Requests`.\
    https://www.rfc-editor.org/rfc/rfc6585

2.  **NGINX --- Limiting Access to Proxied HTTP Resources**\
    Official documentation on request-rate limiting and burst-related
    behavior.\
    https://nginx.org/en/docs/http/ngx_http_limit_req_module.html

3.  **Redis --- Rate Limiting**\
    Official documentation and examples for implementing rate limiting
    with Redis.\
    https://redis.io/docs/latest/develop/use-cases/rate-limiter/

4.  **express-rate-limit --- Documentation**\
    Middleware configuration and usage for Express applications.\
    https://express-rate-limit.mintlify.app/

------------------------------------------------------------------------

## Final Mental Model

Rate limiting is a policy for controlling how frequently a defined
identity may perform a defined action. The algorithm determines how
traffic is counted, how bursts behave, and what state is needed. The
identity key determines which requests count together. In a
multi-instance service, state and updates must be coordinated if the
limit is meant to apply globally. A correct implementation also needs
sensible endpoint policies, careful proxy handling, clear excess-request
behavior, and monitoring.
