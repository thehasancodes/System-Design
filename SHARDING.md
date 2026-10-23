# Database Sharding — Revision Notes

## 1. What is sharding?
**Database sharding** is splitting a dataset into smaller partitions called **shards** and distributing those partitions across multiple database servers. Each shard owns a subset of the overall data. Unlike read replicas, shards do not each hold a complete copy of the entire dataset.

## 2. Why shard?
A single database can eventually face limits in storage, CPU, memory, disk I/O, query throughput, or operational constraints. Sharding can distribute storage and workload across multiple servers.

It adds complexity, so teams commonly optimize queries, add indexes, cache, scale vertically, and consider read replicas before sharding.

## 3. Shard key and routing
A **shard key** is the field used to decide where a record belongs. The application or database infrastructure uses a routing strategy to send a query to the appropriate shard.

A good shard key should distribute workload reasonably well and support common query patterns.

## 4. Common strategies

### A. Range-based sharding
Divide key values into ranges.

| Shard | Example user IDs |
|---|---|
| Shard 1 | 1–100,000 |
| Shard 2 | 100,001–200,000 |
| Shard 3 | 200,001–300,000 |

**Benefit:** Range queries can be efficient when the range aligns with the partitioning.

**Risk:** Some ranges may become much larger or busier than others.

### B. Hash-based sharding
Apply a hash or partitioning function to the shard key. A simplified illustrative rule is:

```text
shard = user_id % number_of_shards
```

With three shards numbered 0, 1, and 2:
- User 101 → Shard 2
- User 102 → Shard 0
- User 103 → Shard 1

Hash-based approaches can distribute records more evenly, but don't guarantee perfect balance. Changing the number of shards can remap many keys unless a more flexible partitioning strategy is used.

### C. Geographic or tenant-based partitioning
Data can be partitioned by region or tenant. This can support locality or isolation requirements, but may create uneven workload if some regions or tenants are much larger than others.

## 5. Read replicas vs. sharding

| Aspect | Read replicas | Sharding |
|---|---|---|
| Data placement | Copies of the same dataset | Different subsets of the dataset |
| Main purpose | Scale reads; can support availability | Distribute storage and workload |
| Typical writes | Routed to the primary | Routed to the shard that owns the record |
| Complexity | Moderate | Often higher |

A system can combine the two: shard the dataset, then add replicas for each shard.

## 6. Challenges
- **Cross-shard queries:** A query may need to run against multiple shards and combine results.
- **Cross-shard joins:** Joins across databases can be expensive or require application-level coordination.
- **Cross-shard transactions:** Coordinating atomic changes across shards is more complex.
- **Hot shards:** Some shards receive much more traffic than others.
- **Rebalancing:** Adding shards may require moving data and updating routing.
- **Shard-key choice:** A poor key can cause uneven distribution or make common queries expensive.
- **Operational complexity:** Monitoring, migrations, backups, and recovery become more involved.

A **scatter-gather** query sends a request to multiple shards and combines their responses.

## 7. Architecture

```text
Application
    |
    v
Shard router / partitioning logic
    |
    +--> Shard A: subset A
    +--> Shard B: subset B
    +--> Shard C: subset C
```

The router may be implemented in application code or provided by database infrastructure, depending on the technology. Adding more database servers to a standard application does not automatically shard its data.

A standard Prisma/PostgreSQL application needs an explicit sharding architecture or suitable supporting infrastructure; merely adding more PostgreSQL instances does not split the dataset automatically.

## 8. When should sharding be considered?
Consider it when evidence shows a single database's capacity or workload is a real bottleneck and simpler techniques are insufficient. Evaluate the shard key against common reads, writes, and joins before committing to a design.

## 9. Quick revision
**Remember:** Read replicas copy the dataset to distribute reads; sharding splits the dataset to distribute storage and workload.

## 10. Self-test
1. What is a shard key?
2. How does range-based sharding differ from hash-based sharding?
3. What is a scatter-gather query?
4. Why can changing the number of shards be difficult?
5. Why is sharding usually more complex than adding a read replica?
