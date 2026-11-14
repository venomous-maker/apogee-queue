# apogee-queue

Queues for [Apogee](https://github.com/venomous-maker): jobs that run later,
on workers.

- **Drivers:** memory, sync, null, Redis and SQL databases.
- **Job features:** retries with backoff, timeouts, middleware (rate limits,
  no overlapping, exception throttling), unique jobs, chains and batches.
- **Around the jobs:** queued event listeners, failed-job storage, and a fake
  for tests.

```grav
import "std"
import "apogee-queue"

class SendWelcome extends Job {
    public userId: long = 0
    constructor(userId: long = 0) { self.userId = userId; self.tries = 3; self.backoff = [10, 60] }
    fn handle(mailer: Mailer, users: UserRepository) { mailer.send(users.find(self.userId), "welcome") }
    fn failed(e: Exception) { alertSomeone(e) }
}

let queue = new QueueManager(container)
queue.addConnection("redis", RedisQueueDriver.open("redis://127.0.0.1:6379/0"))
queue.defaultConnection = "redis"

queue.dispatch(new SendWelcome(42))
queue.dispatch(new SendWelcome(42).onQueue("mail").delay(60))
queue.worker().work("redis", ["mail", "default"], new WorkerOptions())     // what queue:work runs
```

## Jobs

A job extends `Job`:

- **Fields:** a job carries its fields with it, as JSON.
- **`handle()`:** its parameters come from the container, in a scope of the
  job's own.
- **Constructor:** a job needs one that takes no arguments. Literal defaults
  count, as in `constructor(userId: long = 0)`. The worker makes an empty job
  and fills its fields.

**What a field can hold:**

| Kind | Notes |
|---|---|
| Plain values | numbers, text, bools, `T?`, enums |
| Dates and money | `Moment`, `DateTime`, `Decimal` |
| Collections | `Array`/`HashMap` of the plain values above, or `Array<any>`/`HashMap<string, any>` of anything here (objects in a list go in an `Array<any>`: an `Array<Address>` cannot be rebuilt through reflection) |
| Reflected objects | travel field by field |
| Models | travel by id |

A model implements `QueueableEntity` and is fetched fresh on the worker
through `JobCodec.entityResolver`, as Laravel's `SerializesModels` does. A
model that's gone fails the job, or drops it when `deleteWhenMissingModels` is
set. Fields marked `@Transient` stay behind.

**Settings** (fields you set in the constructor, or chain at dispatch):

| Field | What it does |
|---|---|
| `tries` | attempts before the job fails |
| `timeout` | seconds one attempt may take |
| `backoff` | seconds before each retry; the last value repeats |
| `maxExceptions` | how many exceptions fail the job, however many tries are left |
| `retryUntil(moment)` | a deadline after which the job isn't retried |
| `failOnTimeout` | fail at once when an attempt times out |
| queue, connection and delay | `onQueue`, `onConnection`, `delay`, `delayUntil`, `delayFor` |

**Methods a job can override:** `uniqueId()`/`uniqueFor()`,
`uniqueUntilProcessing()`, `middleware()`, `tags()`, `displayName()`,
`failed(e)`.

**Inside `handle()`:**

- `attempts()`, `jobId()`
- `release(seconds)` puts it back on the queue for later.
- `fail(reason)` fails it now.
- `delete()` removes it without running the rest.
- `batch()` is the batch it belongs to.

## Dispatching

```grav
queue.dispatch(job)                       // its id ("" when a unique job with its key is already queued)
queue.dispatchSync(job)                   // now, in this thread
queue.dispatchIf(cond, job)  queue.dispatchUnless(cond, job)  queue.later(30, job)
queue.dispatchAfterResponse(job)          // once the HTTP response is out (the kernel calls runAfterResponse())
queue.bulk(jobs)                          // one RPUSH on Redis, one transaction in SQL
queue.chain([new Fetch(url), new Parse(), new Store()]).onQueue("etl").catch(new ReportFailure()).dispatch()
let b = queue.batch(jobs).name("import").then(new Done()).catch(new Failed()).finally(new Cleanup()).allowFailures().dispatch()
queue.findBatch(b.id).progress()          // 0..100
```

The `sync` connection is the default until you choose another. It runs jobs
as they're dispatched and passes their exceptions to the caller, as Laravel's
does, so jobs work before any queue is set up. Chains and batches work on it
too.

**Batches:**

- `then` runs when every job has succeeded, `catch` on the first failure (which
  cancels the batch unless `allowFailures()`), and `finally` either way.
- Their counts live in a cache store: memory by default, or Redis for many
  machines. The counters are atomic.
- A job can check `batch().cancelled()`, or use the `SkipIfBatchCancelled`
  middleware.

## Drivers

| Driver | For | How |
|---|---|---|
| `MemoryQueueDriver` | tests, one process with worker threads | thread-safe |
| `RedisQueueDriver.open(url, connections)` | many workers on many machines | Laravel's layout: a ready list, delayed and reserved sorted sets |
| `DatabaseQueueDriver(pool)` | SQLite, MySQL/MariaDB, PostgreSQL | `createTable()` makes the jobs table |
| `SyncQueueDriver` | development | runs jobs when dispatched |
| `NullQueueDriver` | switching a queue off | drops every job |

- **Redis:** each pop is one Lua script that moves due delayed and lapsed
  reserved jobs back to the ready list, then reserves the next job. It's
  atomic and takes one round trip. The attempt count travels in a small
  `attempts|id|` prefix, so the payload is never decoded or re-encoded.
  `blockFor` makes an empty pop wait with BLPOP.
- **Database:** a worker claims a job with an `UPDATE` that only succeeds while
  the job is unreserved. Two workers never get the same job, with no row
  locks.
- **Reservations:** a popped job stays reserved for `retryAfter` seconds. If its
  worker dies, the job comes back when the reservation lapses, so a crash
  never loses a job.

## Workers

```grav
let o = new WorkerOptions()
o.tries = 3; o.timeout = 60; o.backoff = [5, 30]; o.sleep = 1.0
o.maxJobs = 1000; o.maxTime = 3600; o.memoryMb = 256; o.stopWhenEmpty = false
o.trapSignals = true; o.output = fn(line: string) { print(line) }
let w = queue.worker()
w.work("redis", ["high", "default"], o)            // queues in priority order
w.workConcurrently(8, "redis", ["default"], o)     // 8 threads in this process
Worker.signalRestart(queue.cache)                  // queue:restart: every worker stops after its job
```

For each job, the worker:

1. Fails it at once when it's out of attempts or past `retryUntil`.
2. Rebuilds it.
3. Runs its middleware and `handle()`, with the timeout. A reusable runner
   thread enforces the timeout instead of a thread per job. A worker whose job
   times out stops with exit code 1, as Laravel's does, so its supervisor
   starts a fresh one.
4. Ends in one of three ways:
   - **Done:** the job is deleted, the next link of its chain is pushed, and
     its batch is counted.
   - **Released:** it goes back on the queue with its backoff.
   - **Failed:** it's recorded, `failed(e)` runs, and so does the chain's or
     batch's catch.

The loop stops on any of these:

- **Signals:** SIGTERM, SIGINT or SIGQUIT, after the job in hand. SIGUSR2
  pauses the worker and SIGCONT resumes it.
- **`queue:restart`.**
- **Limits:** `memoryMb` (exit code 12), `maxJobs`, `maxTime`, or
  `stopWhenEmpty`.

`w.stats` counts processed, failed and released jobs, with average runtime by
job.

## Middleware

```grav
fn middleware() -> Array<JobMiddleware> {
    return [new RateLimitedJob("mailgun", 100, 60) as JobMiddleware,     // 100 a minute; the rest come back later
            new WithoutOverlapping("user:${self.userId}", 10),           // one at a time per user
            new ThrottlesExceptions("crm", 5, 600)]                      // after 5 exceptions, back off 10 minutes
}
```

Middleware uses [grav-cache](https://github.com/venomous-maker/grav-cache)'s
locks and rate limiter, so it holds across workers on a shared store. Also
`SkipIfBatchCancelled` and `SkipUnless(fn)`. Your own implements
`JobMiddleware`.

## Failed jobs

- **Where they're kept:** `queue.failedJobs` is a `MemoryFailedJobStore`,
  `FileFailedJobStore(path)` (one JSON line each) or
  `DatabaseFailedJobStore(pool)`.
- **What's kept:** the payload, connection, queue, exception and time.
- **Managing them:** `queue.retry(id)`, `queue.retryAll()`, plus the store's
  `all`, `find`, `forget` and `flush(olderThanHours)`.

## Queued listeners

`queue.queueListenersOf(events)` sends listener classes whose
`shouldQueue()` is true to the queue, as a `CallQueuedListener` job, instead
of running them during dispatch. The event travels with the job, so its class
needs reflection: give it a decorator such as `@Event`.

## Events

When `queue.events` is set, the queue fires:

| Event | When |
|---|---|
| `JobQueued` | a job is pushed |
| `JobProcessing` | a worker starts a job |
| `JobProcessed` | a job is done (carries `ms`) |
| `JobReleased` | a job goes back on the queue |
| `JobExceptionOccurred` | an attempt throws |
| `JobFailed` | a job fails for good |
| `JobTimedOut` | an attempt times out |
| `WorkerStarting`, `WorkerStopping` | a worker starts or stops |

## Testing

```grav
queue.fake()
code.thatDispatches()
queue.assertPushed::<SendWelcome>()          queue.assertPushed::<SendWelcome>(2)
queue.assertPushedOn::<SendWelcome>("mail")  queue.assertPushedWhere::<SendWelcome>(fn(j: SendWelcome) -> bool { return j.userId == 42 })
queue.assertPushedWithChain::<Fetch>(["Parse", "Store"])
queue.assertNotPushed::<Refund>()            queue.assertNothingPushed()
queue.assertBatched(fn(b: PendingBatch) -> bool { return b.batchName == "import" })
```

## Tests

```
grav test tests/queue.grav
```

The Redis tests need a server at `REDIS_URL` (by default
`redis://127.0.0.1:6379/9`). Under 8 worker threads, 400 Redis jobs take
about 100 ms.
