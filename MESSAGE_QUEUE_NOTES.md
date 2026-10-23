# Message Queues — Notes and Implementation

## 1. What is a message queue?

A **message queue** stores tasks until a consumer (worker) can process them. It separates requesting work from performing that work.

Example: when a user registers, an Express API saves the user and queues a welcome-email job. A separate worker sends the email later, so the HTTP request does not need to wait for the email service.

### Core components

- **Producer:** creates and submits a job. In our example, the Express API.
- **Queue:** holds jobs waiting to be processed.
- **Broker / queue infrastructure:** manages delivery and job state. In our example, Redis stores and coordinates jobs for BullMQ.
- **Consumer / worker:** retrieves jobs and performs the work.
- **Job:** the task name plus its data, such as `send-email` and `{ email: "student@example.com" }`.

### Why use a queue?

- **Responsiveness:** the API can respond after a job is accepted rather than waiting for slow background work.
- **Decoupling:** the API and worker have separate responsibilities and can run as separate processes.
- **Buffering:** jobs can wait when work arrives faster than workers can process them.
- **Retries:** failed jobs can be attempted again when retry behavior is configured.

A queue does not make the email service itself faster, and enqueueing a job does not mean the email has already been sent.

## 2. Job lifecycle

A typical job moves through these states:

1. **Waiting:** queued and waiting for a worker.
2. **Active / processing:** a worker is executing the job.
3. **Completed:** the processor finished successfully.
4. **Failed:** processing failed. Depending on configuration, the job may be retried or eventually remain failed.

An **acknowledgement** is a signal that processing succeeded. Queue systems use their own mechanisms to track this. If a worker crashes at the wrong time, a job may be processed again.

### Retries and idempotency

- **Retry:** try a failed job again.
- **Backoff:** wait before retrying, often increasing the delay.
- **Idempotency:** design an operation so repeating it does not cause unwanted duplicate effects.

A retry can repeat a real-world side effect. For example, an email might have been sent just before a worker crashed. Queues alone do not guarantee that an email or payment happens exactly once.

## 3. Messaging patterns

| Pattern | Main purpose | Example |
|---|---|---|
| Work queue | Distribute jobs among workers | Send emails or generate PDFs |
| Pub/Sub | Broadcast a message to multiple subscribers | Tell active services that data changed |
| Event stream | Retain events for independent consumption and replay | Analytics or event history |

Redis Pub/Sub is transient: subscribers that are offline can miss messages. Use a durable queue or retained stream when messages must be available for later processing.

## 4. Event-Driven Architecture (EDA)

**Event-Driven Architecture** is an approach where components react to events—records that something happened, such as `user.registered` or `assignment.submitted`.

- **Command:** asks for an action, e.g. `SendWelcomeEmail`.
- **Event:** records a fact, e.g. `UserRegistered`.
- **Producer:** publishes or creates the event.
- **Consumer:** reacts to it.

EDA can use queues, Pub/Sub, event streams, direct in-process event handling, or combinations of these. It can reduce coupling, but it also introduces challenges such as duplicate processing, eventual consistency, and debugging across components.

---

# 5. Hands-on implementation: Express + BullMQ + Redis

This lab creates a small queue that simulates sending email. It does **not** send real email and does not require PostgreSQL or Prisma yet.

## Architecture

```text
HTTP client
    |
    v
Express API (producer)
    |
    v
BullMQ queue  <---->  Redis
    |
    v
Worker process (consumer)
    |
    v
Simulated email task
```

The API and worker run in separate Node.js processes. Both connect to the same Redis service and use the same queue name.

## 6. Create the project

In PowerShell, from the directory where you keep your projects:

```powershell
mkdir message-queue-lab
cd message-queue-lab
npm init -y
npm install express bullmq ioredis
```

In this setup, BullMQ's connection configuration uses `ioredis`, so install it explicitly.

Create the folders and files:

```powershell
New-Item -ItemType Directory -Force -Path src, src\queues, src\workers, src\routes
New-Item -ItemType File -Force -Path src\server.js, src\queues\emailQueue.js, src\workers\emailWorker.js
```

The initial structure:

```text
message-queue-lab/
├── src/
│   ├── queues/
│   │   └── emailQueue.js
│   ├── routes/
│   ├── workers/
│   │   └── emailWorker.js
│   └── server.js
├── package.json
└── package-lock.json
```

The `routes` folder is not used in this minimal lab; the route is currently defined in `server.js`.

## 7. Start Redis with Docker

Docker Desktop must be installed and running. Run:

```powershell
docker run -d --name message-queue-redis -p 6379:6379 redis:7
```

Verify the Redis server responds:

```powershell
docker exec -it message-queue-redis redis-cli ping
```

Expected output:

```text
PONG
```

If the container already exists from an earlier run, start it instead of creating it again:

```powershell
docker start message-queue-redis
```

Redis is the service. BullMQ is the Node.js job-queue library that uses Redis.

## 8. Create the queue (producer-side queue definition)

File: `src/queues/emailQueue.js`

```javascript
const { Queue } = require('bullmq');

const emailQueue = new Queue('email-queue', {
  connection: {
    host: '127.0.0.1',
    port: 6379
  }
});

module.exports = emailQueue;
```

- `Queue` is BullMQ's class for adding and managing jobs.
- `email-queue` is the queue name.
- The connection points to Redis exposed on the local machine's port `6379`.
- Exporting the queue lets the API import it.

## 9. Create the Express producer

File: `src/server.js`

```javascript
const express = require('express');
const emailQueue = require('./queues/emailQueue');

const app = express();

app.use(express.json());

app.post('/send-email', async (req, res) => {
  try {
    const { email } = req.body;

    if (!email) {
      return res.status(400).json({
        error: 'Email is required'
      });
    }

    const job = await emailQueue.add(
      'send-email',
      { email },
      {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 1000
        }
      }
    );

    return res.status(202).json({
      message: 'Email job added to queue',
      jobId: job.id
    });
  } catch (error) {
    console.error('Failed to enqueue email:', error.message);
    console.error(error.stack);

    return res.status(500).json({
      error: 'Failed to add email job'
    });
  }
});

const PORT = 3000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Important details

- `emailQueue.add('send-email', { email }, options)` submits a job.
- `'send-email'` is the job name; `{ email }` is the job data.
- `await` waits for BullMQ to complete the enqueue operation.
- HTTP `202 Accepted` means the job was accepted, not that the email was sent.
- `attempts: 3` configures up to three processing attempts in total.
- Exponential backoff starts at a 1,000 ms delay and increases after failures.
- The user should normally be saved successfully before its welcome-email job is added. For systems where losing the event between database commit and queue publication is unacceptable, learn the Transactional Outbox Pattern.

## 10. Create the worker (consumer)

File: `src/workers/emailWorker.js`

```javascript
const { Worker } = require('bullmq');

const emailWorker = new Worker(
  'email-queue',
  async (job) => {
    console.log('Processing job:', job.id);
    console.log('Job name:', job.name);
    console.log('Sending email to:', job.data.email);

    // Simulate the time required to send an email.
    await new Promise((resolve) => setTimeout(resolve, 2000));

    console.log('Email sent successfully!');
  },
  {
    connection: {
      host: '127.0.0.1',
      port: 6379
    }
  }
);

emailWorker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`);
});

emailWorker.on('failed', (job, error) => {
  console.error(`Job ${job?.id} failed:`, error.message);
});

emailWorker.on('error', (error) => {
  console.error('Worker error:', error.message);
});

console.log('Email worker is running...');
```

The worker must use the same queue name, `email-queue`. BullMQ calls the processor function when a job is available. `job.data` contains the payload added by the producer.

The current processor only logs output and waits two seconds; it does not send a real email.

## 11. Run and test the system

Open separate PowerShell terminals in the project directory.

**Terminal 1 — API:**

```powershell
node src/server.js
```

Expected:

```text
Server running on port 3000
```

**Terminal 2 — worker:**

```powershell
node src/workers/emailWorker.js
```

Expected:

```text
Email worker is running...
```

**Terminal 3 — send a test job:**

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:3000/send-email" `
  -Method Post `
  -ContentType "application/json" `
  -Body '{"email":"student@example.com"}'
```

Expected response (the ID may differ):

```text
message                   jobId
-------                   -----
Email job added to queue  1
```

The worker terminal should show the job being processed and then completed.

If you submit a job while the worker is stopped, the job can wait in Redis and be processed when a worker starts, provided the Redis data and job have not been removed or expired.

## 12. Test retry behavior

The queue options already configure three attempts with exponential backoff.

To test retries, temporarily replace the worker processor body with:

```javascript
async (job) => {
  console.log('Processing job:', job.id);
  console.log('Attempt:', job.attemptsMade + 1);

  throw new Error('Simulated email service failure');
}
```

Keep the existing `new Worker(...)` structure and event listeners.

1. Stop the worker with `Ctrl + C`.
2. Save the change and restart the worker.
3. Submit a **new** job through the API.
4. Watch the worker terminal for repeated attempts and failure logs.
5. Restore the successful processor from Section 10 and restart the worker.

`job.attemptsMade + 1` displays the current attempt number because `attemptsMade` counts failed attempts so far. A job configured with `attempts: 3` gets up to three processing attempts total.

## 13. Common problems

### BullMQ says it cannot load `ioredis`

Install the dependency:

```powershell
npm install ioredis
```

Then restart the API and worker processes.

### Redis connection fails

Check the container:

```powershell
docker ps
docker exec -it message-queue-redis redis-cli ping
```

If the container exists but is stopped:

```powershell
docker start message-queue-redis
```

### API returns `Failed to add email job`

Read the detailed error in the API terminal. Check Redis availability, connection settings, and that `ioredis` is installed.

### A job doesn't process

Confirm the worker process is running and uses the exact same queue name and Redis connection as the producer.

---

# 14. Essential production lessons

1. **Queue acceptance is not task completion.** Track job status separately if clients need to know the result.
2. **Plan for retries and duplicates.** A job can run more than once; make handlers idempotent where possible.
3. **Handle permanent failures.** Retries cannot fix invalid email addresses or other permanent errors. Production systems should inspect exhausted jobs and decide how to alert or recover.
4. **Protect the queue endpoint.** Validate input, authenticate callers where appropriate, and rate-limit public endpoints.
5. **Shut down cleanly.** Production APIs and workers should close their queue/worker connections during graceful shutdown.
6. **Plan for database/queue consistency.** A database transaction and a separate queue write are not automatically atomic. Use an outbox when the reliability requirement justifies it.
7. **Do not expose secrets.** Use environment variables or a secrets manager for credentials in real deployments. The local lab uses no secrets.
8. **Redis persistence and availability matter.** A successful local `PONG` proves Redis is responding, not that the system has production-grade durability or high availability.

## 15. Quick revision

- A **producer** adds jobs.
- A **queue** holds and coordinates jobs.
- A **worker/consumer** processes jobs.
- **Redis** is the backend used by BullMQ in this lab.
- **BullMQ** provides job queue behavior and APIs.
- **Retries** help recover from temporary failures.
- **Idempotency** helps make repeated processing safe.
- **Pub/Sub** broadcasts messages; a work queue distributes jobs; an event stream retains events for replay.
- **EDA** is an architectural pattern in which components react to events.

## References

- BullMQ documentation: https://docs.bullmq.io/
- BullMQ retries: https://docs.bullmq.io/guide/retrying-failing-jobs
- Redis job queue use case: https://redis.io/docs/latest/develop/use-cases/job-queue/
- Redis Pub/Sub: https://redis.io/docs/latest/develop/use-cases/pub-sub/
- Redis Streams: https://redis.io/docs/latest/develop/use-cases/streaming/
