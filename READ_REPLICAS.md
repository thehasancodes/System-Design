# Database Read Replicas — Revision Notes

## 1. What is a read replica?
A **read replica** is a database copy that receives replicated data from a primary database and is commonly used to serve read queries.

In a typical primary-replica setup:
- The **primary** handles writes and may also handle reads.
- **Read replicas** serve eligible reads.
- Replication carries changes from the primary to replicas.

A replica does not automatically receive application traffic just because it exists. The application, ORM, driver, or database proxy must be configured to route queries appropriately.

## 2. Why use read replicas?
Suppose a service receives 10,000 database requests per second: 8,000 reads and 2,000 writes. If reads are a bottleneck, replicas can distribute eligible read work across more database servers. The exact improvement depends on query cost, hardware, network, replication, and routing.

## 3. How replication works
1. The primary commits a change.
2. The database transmits changes using its replication mechanism, often based on transaction-log records.
3. A replica receives and applies the changes.
4. Queries routed to the replica can see the updated data once it has been applied.

The database usually does not copy the entire dataset every time a single row changes.

## 4. Replication lag and stale reads
**Replication lag** is the delay between a change being committed on the primary and becoming visible on a replica.

Example:
1. A user changes their name from `Alex` to `Sam`.
2. The primary commits the update.
3. The application immediately reads from a replica.
4. The replica has not applied the change yet, so it returns `Alex`.

This is a **stale read**. The write may have succeeded even though the replica still returns the old value.

Possible strategies:
- Read from the primary when the latest committed value is required immediately.
- Route a user's reads to the primary for a period after a write.
- Use database-supported consistency or replication-aware routing.
- Accept replica lag where a small delay is acceptable.

## 5. Good use cases and cautions
**Often suitable:** product catalogues, news and content browsing, analytics/reporting, read-heavy applications.

**Use caution:** immediately displaying a changed profile, inventory decisions where stale data could cause overselling, or balance/status checks requiring the latest committed value.

A replica doesn't automatically solve slow SQL, missing indexes, or a write-heavy bottleneck.

## 6. Read replicas vs. other techniques

| Technique | What it does |
|---|---|
| Connection pooling | Reuses connections and controls connection usage |
| Read replicas | Copies data to distribute eligible reads |
| Caching | Avoids some queries by reusing stored results |
| Sharding | Splits data into separate partitions |
| Backups | Preserve recoverable copies for restoration |

A read replica is not necessarily a backup. Accidental deletions or corrupt changes may also replicate, so backups are still needed.

## 7. Combining replicas with sharding
A system can split data into shards and maintain replicas for each shard. Sharding distributes different portions of data; replicas copy the data belonging to a primary or shard.

## 8. Quick revision
**Remember:** Read replicas distribute read workload by maintaining copies of primary data. Replication lag can cause stale reads, and the application must route queries deliberately.

## 9. Self-test
1. What problem do read replicas primarily help with?
2. What is replication lag?
3. Why might a user see an old profile value immediately after an update?
4. Why is a read replica not a substitute for backups?
