# Connection Pooling — Revision Notes

## 1. What is connection pooling?
A **database connection pool** is a managed set of reusable database connections. Instead of opening a fresh connection for every API request, an application borrows an available connection, runs its query, and returns the connection to the pool.

Creating connections has overhead, including network setup, authentication, session initialization, and resource allocation.

## 2. Why it matters
Without pooling, a busy API may repeatedly create connections and overwhelm the database with connection overhead. With a pool:
1. The application maintains a bounded number of connections.
2. A request asks the pool for a connection.
3. The query executes.
4. The connection is released for reuse.

Returning a connection to the pool usually does **not** close the underlying database connection.

## 3. What if all connections are busy?
If a pool has 10 connections and all 10 are in use, another request generally waits until one becomes available. If the acquisition timeout expires, the operation can fail. A pool bounds connection usage; it does not make the database execute unlimited queries simultaneously.

## 4. Important configuration terms
- **Maximum pool size:** Maximum connections managed by the pool.
- **Acquisition timeout:** How long a request waits to get a connection.
- **Idle timeout:** How long an unused connection may remain open.
- **Query duration:** How long a connection remains occupied by a query or transaction.

A larger pool is not always better. The database has a total connection budget, and each application instance may consume part of it.

## 5. Node.js example with PostgreSQL
Using the `pg` package:

```javascript
const { Pool } = require("pg");

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});

async function getUsers() {
  const result = await pool.query(
    "SELECT id, name FROM users"
  );
  return result.rows;
}
```

Create the pool once per application process and reuse it; don't create a new pool for every request.

For a multi-statement SQL transaction, acquire one client and use that same client for `BEGIN`, all statements, and `COMMIT` or `ROLLBACK`. Separate `pool.query()` calls are not guaranteed to use the same connection.

## 6. Prisma and deployment
Prisma manages database connections according to its version, driver/adapter, and configuration. Reuse the application's Prisma client rather than repeatedly constructing clients for each request.

With multiple backend instances, each instance may have its own pool:
- 5 instances × 20 connections each = up to 100 connections.
- 20 instances × 20 connections each = up to 400 connections.

Plan pool sizes around the database's actual connection limit. A pooler such as PgBouncer may help in suitable PostgreSQL deployments.

## 7. Related concepts

| Technique | Main purpose |
|---|---|
| Connection pooling | Reuse connections and bound connection usage |
| Caching | Avoid repeating some database queries |
| Read replicas | Distribute eligible read queries |
| Sharding | Split data across database partitions |
| Load balancing | Distribute requests across application instances |

## 8. Common mistakes
- Creating a new pool for each request.
- Opening more connections than the database can support.
- Holding a connection while doing slow, unrelated work.
- Forgetting to release a checked-out client in a `finally` block.
- Assuming pooling increases database capacity without limit.

## 9. Quick revision
**Remember:** A pool reuses a bounded number of database connections. Requests may wait when the pool is fully occupied. Pool sizes must be considered across all application instances.

## 10. Self-test
1. Why is creating a new database connection for every request expensive?
2. What happens when every connection in a pool is busy?
3. Why can scaling from 5 to 20 backend instances increase connection pressure?
4. Why should all statements in a transaction use the same database client?
