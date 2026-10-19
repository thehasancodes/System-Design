# Caching --- Complete Learning Notes

## Project Learn CSE in Depth

> **Learning approach:** Understand the problem first, then the
> mechanisms, trade-offs, and practical design decisions. These notes
> consolidate the caching material covered so far. The five-batch plan
> is included at the end; only Batches 1 and 2 have been studied in
> detail so far.

------------------------------------------------------------------------

## Table of Contents

1.  [Why Caching Exists](#1-why-caching-exists)
2.  [Core Vocabulary](#2-core-vocabulary)
3.  [Cache Hits, Misses, and the Basic
    Economics](#3-cache-hits-misses-and-the-basic-economics)
4.  [Freshness: TTL and Invalidation](#4-freshness-ttl-and-invalidation)
5.  [Where a Cache Can Live](#5-where-a-cache-can-live)
6.  [Device-Side Caching](#6-device-side-caching)
7.  [In-Process Node.js Caching](#7-in-process-nodejs-caching)
8.  [Redis: A Shared Server-Side
    Cache](#8-redis-a-shared-server-side-cache)
9.  [PostgreSQL and Operating-System
    Caches](#9-postgresql-and-operating-system-caches)
10. [CDNs and Reverse-Proxy Caches](#10-cdns-and-reverse-proxy-caches)
11. [Compare Cache Locations](#11-compare-cache-locations)
12. [Choosing What to Cache](#12-choosing-what-to-cache)
13. [How Caching Fits the Current Backend
    Stack](#13-how-caching-fits-the-current-backend-stack)
14. [Common Failure Modes and Design
    Warnings](#14-common-failure-modes-and-design-warnings)
15. [Knowledge Check](#15-knowledge-check)
16. [Five-Batch Learning Roadmap](#16-five-batch-learning-roadmap)
17. [References](#17-references)

------------------------------------------------------------------------

## 1. Why Caching Exists

Imagine the FoCT Connect app has an endpoint:

`GET /api/programmes`

The endpoint retrieves a list of department programmes from PostgreSQL
through Prisma. If hundreds of students request the same list, and the
list rarely changes, the backend may repeatedly perform almost identical
work.

A database query is not just a disk read. Depending on the query, the
database may need to plan or execute the query, locate rows or index
entries, apply visibility rules, build a result, and return it to the
application. Even when relevant database pages are already in memory,
the query still has to be executed.

**Caching** stores a reusable copy of data or a result so a later
operation can reuse it rather than repeat some or all of the original
work.

A cache can store: - Data records, such as a list of programmes. - A
computed result, such as an aggregated dashboard summary. - An HTTP
response or static asset. - A local copy of previously downloaded
information.

Caching is a *role or technique*, not the name of a single technology.
Redis is one possible cache technology; it is not synonymous with
caching.

### The central trade-off

Caching can reduce latency and repeated work, but introduces another
copy of the data. That copy costs memory and must be managed when the
authoritative data changes.

> **Mental model:** Caching exchanges some storage and consistency
> complexity for less repeated work.

------------------------------------------------------------------------

## 2. Core Vocabulary

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  **Cache**                           A store of reusable copies of data
                                      or results.

  **Cache key**                       The identifier used to look up a
                                      cached entry,
                                      e.g. `programmes:all`.

  **Cache value**                     The stored data associated with a
                                      key.

  **Cache hit**                       The requested entry exists and can
                                      be used.

  **Cache miss**                      The requested entry is absent or
                                      unusable, so the application must
                                      obtain the result elsewhere.

  **TTL (time to live)**              How long an entry is allowed to
                                      remain valid under the cache's
                                      expiration policy.

  **Expiration**                      The point at which an entry is
                                      considered no longer usable under
                                      its TTL policy.

  **Invalidation**                    Making a cached entry unusable or
                                      removing it, commonly after the
                                      underlying data changes.

  **Eviction**                        Removing entries to make room or
                                      comply with a memory policy.
                                      Eviction can happen before an
                                      application's intended TTL.

  **Authoritative store**             The system treated as the source of
                                      truth for the data; in this
                                      backend, normally PostgreSQL.

  **Stale data**                      A cached copy that no longer
                                      reflects the latest authoritative
                                      value.

  **Cache-aside**                     A pattern in which application code
                                      checks the cache first, loads from
                                      the source on a miss, and then
                                      populates the cache.
  -----------------------------------------------------------------------

TTL, invalidation, and eviction are related but not identical: - TTL
expresses how long an entry may be reused. - Invalidation responds to a
known change or event. - Eviction is a storage-management action that
frees cache capacity.

------------------------------------------------------------------------

## 3. Cache Hits, Misses, and the Basic Economics

A common cache-aside read flow is:

1.  Look up the requested key in the cache.
2.  If a usable entry exists, return it (**hit**).
3.  Otherwise, query the authoritative data store (**miss**).
4.  Store the result in the cache.
5.  Return the result.

### Simplified example

Suppose 100 requests ask for the same programme list, and the data does
not change during the example.

-   Without an application cache: up to 100 database queries.
-   With a cache that misses on the first request and hits on the next
    99: 1 database query and 99 cache hits.

In this simplified example, database-query count falls by 99%. That is
**not** a promise of a 99% response-time improvement. Actual results
depend on cache-hit rate, query cost, network latency, serialization,
payload size, and other overhead.

### A cache is not automatically faster

Consider purely illustrative timings: - Database-backed request: 12
ms. - Cache lookup on a hit: 2 ms. - Cache miss that queries the
database and writes the cache: 14 ms.

A hit can be cheaper, but a miss can cost more than the original
database-backed request because the application performs extra work.
These numbers are examples, not benchmarks.

Caching is most useful when: - The same information is requested
repeatedly. - The original computation or retrieval is relatively
expensive. - The result remains valid long enough to be reused. - The
cost of cache access is lower than the work it avoids.

Caching may be unhelpful when data is rarely requested, changes
constantly, is cheap to compute, or has so many distinct keys that
entries are seldom reused.

------------------------------------------------------------------------

## 4. Freshness: TTL and Invalidation

Suppose an administrator updates a department phone number in
PostgreSQL. Redis or an in-process cache may still contain the old
number. The next student request can receive that stale value unless the
cache is refreshed, invalidated, or allowed to expire.

Two important strategies are:

### TTL-based expiration

Each entry is given a maximum reuse period. When the TTL expires, the
application should treat the entry as unavailable and load fresh data on
a subsequent request.

**Benefit:** Old entries eventually stop being reused even if no
explicit invalidation happens.

**Limitation:** TTL does not guarantee immediate freshness. If a value
changes just after the cache is populated, the old value may remain
usable until expiration.

### Explicit invalidation

When the underlying data changes, application logic removes or refreshes
related cached entries.

**Benefit:** A known update can be reflected sooner than waiting for the
TTL.

**Limitation:** Every relevant update path must handle invalidation
correctly. If an update path forgets to invalidate a key, stale data may
persist.

These techniques can be combined. Neither eliminates every consistency
problem, especially when several application instances or systems are
involved.

### Cache stampede (introductory concept)

If a popular entry expires and many requests miss at nearly the same
time, those requests may all query the database to rebuild the same
value. This burst of duplicated work is commonly called a **cache
stampede**. More advanced handling is part of the later learning plan.

------------------------------------------------------------------------

## 5. Where a Cache Can Live

The cache's location determines its speed, sharing behavior, lifetime,
and failure characteristics.

  -----------------------------------------------------------------------
  Location                            Typical purpose
  ----------------------------------- -----------------------------------
  User's device/browser               Reuse data without another network
                                      request; support responsiveness and
                                      offline use.

  Node.js process memory              Reuse values inside one running
                                      backend process.

  Redis or another shared cache       Share entries among multiple
  service                             backend instances.

  PostgreSQL memory buffers           Keep database pages in memory to
                                      reduce physical reads.

  Operating-system file cache         Keep recently used file data in
                                      memory.

  CDN/reverse proxy                   Serve eligible responses before
                                      requests reach the origin backend.

  CPU caches                          Hardware-level caching; useful
                                      background context, but usually not
                                      an application-level design
                                      decision.
  -----------------------------------------------------------------------

The word *cache* refers to the function; the storage medium and location
can vary.

------------------------------------------------------------------------

## 6. Device-Side Caching

Device-side caches are close to the user and can avoid a network round
trip.

### Common forms

-   **In-memory app state:** React state or a state-management store.
    Fast, but normally lost when the process restarts.
-   **Persistent local storage:** For example, React Native
    AsyncStorage. Data can survive app restarts, but the application
    must manage expiration and updates.
-   **Local database:** SQLite can store structured data for offline
    reading, filtering, and queries.
-   **Browser HTTP cache:** The browser can reuse eligible responses
    according to HTTP caching rules.

### Example flow

1.  The app fetches the programme list.
2.  The app saves a local copy.
3.  The student returns to the screen.
4.  The app displays the saved copy quickly and, if appropriate,
    refreshes it from the server.

Saving data locally does not automatically implement this behavior; the
app must choose when to read, display, refresh, and replace cached data.

### Good uses

-   Previously loaded notes or programme lists.
-   Screens that should reopen quickly.
-   Downloaded study materials that must be available offline.
-   Static assets such as images or versioned JavaScript bundles.

### Security warning

Local storage is not automatically safe just because it is on the user's
phone. Do not treat cached content as proof of authorization. Be
especially careful with sensitive data and account switching on shared
devices.

------------------------------------------------------------------------

## 7. In-Process Node.js Caching

A simple in-process cache can use a JavaScript `Map`:

``` js
const cache = new Map();
```

A key such as `programmes:all` can map to a programme-list value. The
route checks the map first, queries PostgreSQL on a miss, and stores the
result.

### Advantages

-   Very low access overhead: data is already in the process's memory.
-   Simple to prototype and understand.
-   No separate cache service is required.

### Limitations

-   **Process-local:** Each Node.js process has its own memory.
-   **Lost on restart:** A restart normally destroys the entries.
-   **Memory limits:** Cached objects compete with the application for
    RAM.
-   **Manual policy:** TTL, eviction, and invalidation need to be
    implemented or supplied by a cache library.
-   **Cross-instance inconsistency:** Multiple backend instances can
    hold different values.

### Two-instance example

If Express instance A and instance B each have their own `Map`, an entry
written to A does not automatically appear in B. If A invalidates its
copy after an update, B may still return its stale copy.

Use in-process caching for small, frequently reused data when
process-local copies are acceptable. It is not a shared distributed
cache.

------------------------------------------------------------------------

## 8. Redis: A Shared Server-Side Cache

Redis is normally deployed as a separate server-side service. Multiple
Express instances can connect to the same Redis deployment and reuse
shared entries.

### Cache-aside flow

1.  Express asks Redis for a value.
2.  On a valid hit, Express returns it.
3.  On a miss, Express queries PostgreSQL through Prisma.
4.  Express stores the result in Redis, commonly with a TTL.
5.  Later requests can reuse the cached value.

### Why Redis instead of a `Map`?

A shared Redis service allows multiple backend instances to use the same
cache entries. With separate in-process maps, each instance maintains
its own copy.

### Trade-offs

-   Redis requires a network request, unlike an in-process memory
    lookup.
-   It adds infrastructure, configuration, monitoring, and
    failure-handling complexity.
-   A shared cache does not automatically stay synchronized with
    PostgreSQL.
-   Entries can expire or be evicted.
-   Cache data should generally be treated as rebuildable when
    PostgreSQL is the authoritative store.

Redis can be configured for persistence, but a typical cache-aside
design should not rely on a cached copy as the only durable copy of
important application data.

### When it is worth considering

-   Repeated database reads are a measured bottleneck.
-   Many requests reuse the same data.
-   Several backend instances need shared cached results.
-   The data can tolerate a clearly defined freshness policy.

Redis should be added because measurements and requirements justify it,
not merely because it is popular.

------------------------------------------------------------------------

## 9. PostgreSQL and Operating-System Caches

PostgreSQL uses `shared_buffers` to hold database pages in memory. The
operating system also maintains a file-system cache that PostgreSQL can
benefit from.

Consider:

``` sql
SELECT * FROM programmes WHERE id = 10;
```

Even if the required page is already in memory, PostgreSQL still has to
execute the query and identify the matching result. In-memory pages can
avoid physical disk reads, but they do not mean that the query
disappears.

### Internal database cache vs application-level cache

  -----------------------------------------------------------------------
  Situation                           Work potentially avoided
  ----------------------------------- -----------------------------------
  Relevant PostgreSQL pages are in    Some physical disk reads
  memory                              

  Valid result is in Redis            The PostgreSQL query can be skipped

  Valid result is in Node.js memory   Redis access and the database query
                                      can be skipped

  Valid result is on the device       The backend request may be skipped
                                      altogether
  -----------------------------------------------------------------------

This is a simplified comparison; actual performance depends on query
complexity, indexes, payload size, network latency, and other factors.

**Key distinction:** PostgreSQL's internal caches help the database
access its pages efficiently. An application-level cache can avoid
asking the database for a result at all.

Do not assume Redis is always faster overall. Measure the actual query
and end-to-end request path.

------------------------------------------------------------------------

## 10. CDNs and Reverse-Proxy Caches

A **CDN (Content Delivery Network)** can serve cached content from
locations closer to users. A reverse proxy can also cache eligible
responses before forwarding requests to the origin server.

For example, a CDN can serve a versioned JavaScript bundle to many users
without every request reaching Express. That reduces origin traffic and
can reduce latency.

Useful candidates include: - Static assets. - Public content. -
Carefully designed read-only endpoints.

### Important caution

A shared cache must not accidentally serve one student's private
response to another student. Cache keys and HTTP caching directives must
account for whether the response is public or user-specific.

------------------------------------------------------------------------

## 11. Compare Cache Locations

  ------------------------------------------------------------------------
  Cache location    Shared across      Usually survives  Main work it may
                    users/instances?   restart?          avoid
  ----------------- ------------------ ----------------- -----------------
  Device memory     No, generally one  No                Local
                    app instance                         recomputation or
                                                         refetching

  Device persistent Usually one        Yes               Network fetches
  storage           device/app                           when cached data
                                                         is usable

  Node.js `Map`     One process        No                Repeated backend
                                                         work and database
                                                         queries

  Redis             Yes, across        Depends on        Repeated database
                    connected          configuration     queries
                    instances                            

  PostgreSQL        Within the         Not a guaranteed  Some physical
  buffers           database server    durable cache     disk reads

  CDN/reverse proxy Often shared       Depends on        Requests reaching
                    across clients     service           the origin server
  ------------------------------------------------------------------------

"Survives restart" describes the cached copy, not the authoritative
database data.

------------------------------------------------------------------------

## 12. Choosing What to Cache

### Likely candidates

-   Public programme lists that change infrequently.
-   Semester or section lists.
-   Read-heavy reference data.
-   Expensive computations whose results can be reused.
-   Static assets.

### Candidates needing extra care

-   Student-specific dashboard data.
-   Notifications and read/unread status.
-   Permission-dependent responses.
-   Frequently updated records.
-   Data containing sensitive information.

For user-specific results, the cache key and authorization flow must
prevent cross-user data leakage. A key that ignores the user or relevant
permission context can return the wrong person's data.

### Practical decision sequence

1.  Measure where the latency comes from.
2.  Check whether repeated work exists.
3.  Determine how fresh the result must be.
4.  Choose a cache location that fits the sharing and lifetime
    requirements.
5.  Define keys, TTL, invalidation, and failure behavior.
6.  Measure hit rate, latency, memory usage, and database load after
    introducing the cache.

------------------------------------------------------------------------

## 13. How Caching Fits the Current Backend Stack

The current backend direction is:

-   JavaScript / Node.js
-   Express
-   PostgreSQL
-   Prisma
-   Redis planned for a later phase

A sensible learning and implementation progression is:

1.  Understand the request and query costs first.
2.  Learn cache-aside using a simple example.
3.  Understand keys, TTL, invalidation, and account isolation.
4.  Decide whether a local in-process cache or shared Redis is
    appropriate for the actual requirement.
5.  Add Redis only when the use case and measurements justify the extra
    dependency.

Redis is optional infrastructure, not a mandatory component of every
Express + Prisma backend.

------------------------------------------------------------------------

## 14. Common Failure Modes and Design Warnings

-   **Stale values:** The authoritative data changed but the cached copy
    did not.
-   **Incorrect cache key:** Different queries or users accidentally map
    to the same entry.
-   **Unbounded memory:** In-process cache entries accumulate without
    expiration or eviction.
-   **Cache stampede:** Many simultaneous misses trigger duplicate
    expensive work.
-   **Cold cache:** After restart or expiration, requests must rebuild
    entries.
-   **Cache outage:** Application behavior must be considered if Redis
    is unavailable.
-   **Sensitive data leakage:** Shared entries may cross user or
    permission boundaries if keys and access checks are poorly designed.
-   **Caching the wrong layer:** A cache cannot compensate for every
    inefficient query or poor database index.
-   **Premature complexity:** A cache can make correctness and
    operations harder without delivering meaningful benefits.

------------------------------------------------------------------------

## 15. Knowledge Check

1.  **Two Express instances each use their own JavaScript `Map`. Will an
    entry added to A automatically appear in B?**\
    **Answer:** No. Their process memory is separate.

2.  **PostgreSQL has the relevant table pages in memory. What might
    Redis still save on a valid cache hit?**\
    **Answer:** The application can skip executing the PostgreSQL query.

3.  **Several Express instances must share a cached programme list. What
    kind of cache fits?**\
    **Answer:** A shared Redis service they all connect to.

4.  **A student needs downloaded notes offline. Which location is
    directly useful?**\
    **Answer:** Persistent storage on the device.

5.  **Does a TTL guarantee that a cached value is always perfectly
    current?**\
    **Answer:** No. It bounds the intended reuse period but does not
    provide instantaneous consistency.

6.  **Is Redis automatically synchronized with PostgreSQL?**\
    **Answer:** No. The application must design cache population,
    expiration, and invalidation.

------------------------------------------------------------------------

## 16. Five-Batch Learning Roadmap

The topic is intentionally split into exactly five batches so each
section can be studied carefully.

  -----------------------------------------------------------------------
  Batch                   Topic                   Status
  ----------------------- ----------------------- -----------------------
  1                       Foundations and the     **Covered**
                          Problem It Solves:      
                          cache purpose,          
                          hits/misses,            
                          performance trade-offs, 
                          freshness               

  2                       Cache Architecture and  **Covered**
                          Storage: device,        
                          process memory, Redis,  
                          database caches, CDN    

  3                       Cache Keys, TTL,        Planned
                          Eviction, and           
                          Invalidation            

  4                       Cache-Aside and Redis   Planned
                          Integration in          
                          Node.js/Express         

  5                       Production Design:      Planned
                          consistency, stampedes, 
                          failure handling,       
                          security, measurement,  
                          and design decisions    
  -----------------------------------------------------------------------

The future batches are a roadmap, not claims that those lessons have
already been completed.

------------------------------------------------------------------------

## 17. References

Official and primary documentation:

1.  **Redis --- Cache-aside pattern**\
    https://redis.io/docs/latest/develop/use-cases/cache-aside/

2.  **Redis --- Cache-aside with Node.js**\
    https://redis.io/docs/latest/develop/use-cases/cache-aside/nodejs/

3.  **PostgreSQL --- Resource Consumption Configuration
    (`shared_buffers`)**\
    https://www.postgresql.org/docs/current/runtime-config-resource.html

4.  **MDN --- HTTP Caching**\
    https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching

5.  **MDN --- `Cache-Control`**\
    https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control

------------------------------------------------------------------------

## Final Mental Model

A cache stores a reusable copy of data or a result to reduce repeated
work. Its location determines who can access it and what work it can
avoid. Its key determines what result is retrieved; TTL and invalidation
influence freshness; eviction manages finite storage. A useful cache
must improve the measured system without violating correctness,
authorization, or freshness requirements.
