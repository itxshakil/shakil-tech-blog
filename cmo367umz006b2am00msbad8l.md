---
title: "The Audit Trail: Building a System That Remembers"
datePublished: 2026-04-17T17:17:08.703Z
cuid: cmo367umz006b2am00msbad8l
slug: audit-trail-that-remembers
canonical: https://blog.shakiltech.com/laravel-audit-trail-building-a-system-that-remembers/
cover: https://cdn.hashnode.com/uploads/covers/63bac8d8624134a0c9b53b97/933e6546-d3dc-4031-a3cf-d03a4a2f55e5.png
tags: laravel, php

---

A transaction record had been modified.

The amount was different from what the user had submitted. Support escalated it. The user denied changing anything. The developer who last touched the code was on leave. We looked at the database — the record had an `updated_at` from two days ago and a different value than expected. That was everything we had.

No who. No which fields. No context about what request caused it.

We had a working application. We did not have a system that remembered anything.

That was the incident that made us build this.

---

## What you will build

- A request ID generated in middleware that automatically appears in every log line for that request — no manual threading required
- Field-level model diffs that capture what changed *from* and *to*, not just that a record was updated
- Append-only log files segmented to stay fast at millions of records
- A Gate hook that logs failed permission checks before they become incidents

---

## What an audit trail actually needs to prove

Before writing code, it is worth being precise about what you are building — because "audit trail" means different things in different contexts, and the requirements determine the architecture.

If you need a queryable, database-backed audit log with a ready-made API, [spatie/laravel-activitylog](https://github.com/spatie/laravel-activitylog) is the standard choice and it is well built. What follows is for when your requirements go further — append-only guarantees, field-level diffs, and audit records that are genuinely hard to tamper with.

In a compliance-heavy environment — fintech, healthcare, any regulated domain — an audit trail needs to answer four questions about any data change:

- **Who** made the change (user ID, IP address, user agent)
- **What** changed — not "the record was updated," but which fields, from what value to what
- **When** it changed
- **Why it is trustworthy** — the audit record itself cannot be quietly modified

Most implementations get the first three. The fourth is where they fail, and it is the one that matters most in an actual audit.

Here is the problem with a database audit table: your application code writes to it. That means your application code can also `UPDATE` it. A table that application code can modify is not an immutable record — it is a mutable history. An auditor who understands this will ask how you prevent tampering, and "we trust our own code" is not a satisfying answer.

This shapes the decision to use file-based logging. But first, there is a more foundational problem: correlation.

---

## The request ID: one thread through every log

A model change does not happen in isolation. It happens during a request — a specific HTTP call from a specific user at a specific moment. Without connecting the model change to that request, you have timestamped facts with no story between them.

The solution is a request ID: a UUID generated at the start of every request and written into every log line that request produces.

Before building the middleware class, add the `request` and `query` channels to `config/logging.php`:

```php
// config/logging.php
'channels' => [

    // ... your existing channels ...

    'request' => [
        'driver' => 'daily',
        'path'   => storage_path('logs/request.log'),
        'level'  => 'debug',
        'days'   => 90,        // Retain 90 days — adjust to your compliance requirement
    ],

    'query' => [
        'driver' => 'daily',
        'path'   => storage_path('logs/query.log'),
        'level'  => 'debug',
        'days'   => 30,
    ],

],
```

Then the middleware:

```php
class RequestLogger
{
    public const HEADER_NAME          = 'X-Request-Id';
    public const REQUEST_ID_ATTRIBUTE = 'request_id';

    public function handle(Request $request, Closure $next): Response
    {
        $requestId = (string) Str::uuid();

        // Store in request attributes for internal access within the same request
        $request->attributes->set(self::REQUEST_ID_ATTRIBUTE, $requestId);
        // Also set as a header — downstream systems and the browser can read it
        $request->headers->set(self::HEADER_NAME, $requestId);

        // shareContext injects request_id into every Log:: call
        // for the rest of this request automatically — no manual threading required
        Log::shareContext(['request_id' => $requestId]);

        $startedAt = hrtime(true); // Monotonic clock — more accurate than microtime()
        $response  = $next($request);
        $response->headers->set(self::HEADER_NAME, $requestId);

        $this->logRequest($request, $response, $requestId, $startedAt);

        return $response;
    }

    private function logRequest(
        Request $request,
        Response $response,
        string $requestId,
        int $startedAt
    ): void {
        Log::channel('request')->info('request.completed', [
            'request_id'  => $requestId,
            'method'      => $request->method(),
            'path'        => $request->getPathInfo(),
            'status'      => $response->getStatusCode(),
            'duration_ms' => round((hrtime(true) - $startedAt) / 1_000_000, 2),
            'user_id'     => $request->user()?->getAuthIdentifier(),
            'ip'          => $request->ip(),
        ]);
    }
}
```

Register it as global middleware:

```php
// Laravel 11 — bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\App\Http\Middleware\RequestLogger::class);
})

// Laravel 10 and earlier — app/Http/Kernel.php
protected $middleware = [
    // ...
    \App\Http\Middleware\RequestLogger::class,
];
```

Every log entry your application writes from this point on will now include the request ID automatically. Here is what a request log entry looks like in `storage/logs/request.log`:

```json
{
  "message": "request.completed",
  "request_id": "9d4e2f1a-83bc-4a7c-b291-7e5f3d9a1c84",
  "method": "PATCH",
  "path": "/transactions/1101",
  "status": 200,
  "duration_ms": 43.2,
  "user_id": 42,
  "ip": "192.168.1.10",
  "level": "info",
  "level_name": "INFO"
}
```

And a model change log entry in `storage/logs/activities/transactions/1100/transaction_1101.log`:

```json
{
  "event": "updated",
  "request_id": "9d4e2f1a-83bc-4a7c-b291-7e5f3d9a1c84",
  "time": "2024-03-15 14:32:07",
  "model_id": 1101,
  "changes": {
    "amount": { "old": "5000.00", "new": "500.00" }
  },
  "user_id": 42,
  "ip": "192.168.1.10",
  "user_agent": "Mozilla/5.0 ..."
}
```

The same `request_id` appears in both files. That is the correlation. When a support ticket says "something changed on Transaction 1101," you grep that ID across both logs and have the complete picture immediately.

**Why `Log::shareContext()` instead of passing the ID manually everywhere?** [`shareContext()`](https://laravel.com/docs/logging#contextual-information) injects the given data into every `Log::` call for the rest of the request lifecycle automatically. You do not thread the request ID through your services, model observers, or Gate hooks. Whatever you log, the request ID is already there.

**Why UUID instead of a random integer?** A six-digit random int has a real collision probability under concurrent load. A UUID is 128 bits of randomness — collision is not a realistic concern. The ID also goes into the response header (`X-Request-Id`), so a user who raises a support ticket can include it and you find the exact request in seconds.

**Why a single structured entry after the response rather than separate request-in and response-out entries?** One entry per request gives you the complete picture — method, path, status code, duration — in one line without cross-referencing. The `hrtime(true)` monotonic clock is more accurate for duration measurement than `microtime()` because it is not affected by system clock adjustments.

**A note on `X-Forwarded-For`:** When your application sits behind a load balancer, `request()->ip()` returns the proxy's IP, not the user's. The `TrustProxies` middleware resolves this — once configured, `request()->ip()` returns the real client IP. Pass it to external API calls so downstream system logs also record the real user, not your server's address.

---

## Slow queries: while you are here, add this

This is a separate concern from the audit trail — but since you are already setting up logging infrastructure, it costs almost nothing to add and consistently pays off the first time a performance problem hits production.

The idea: log slow queries with severity levels matched to how serious the slowness actually is.

```php
// AppServiceProvider.php
// Scope this to non-production environments or behind a config flag
// in high-traffic systems — DB::listen fires on every query.
if (config('app.debug') || config('logging.query_listener')) {
    DB::listen(function ($query) {
        $time = $query->time;

        match(true) {
            $time > 10000 => Log::channel('query')->critical("Extremely slow ({$time}ms)"),
            $time > 1000  => Log::channel('query')->error("Very slow ({$time}ms)"),
            $time > 100   => Log::channel('query')->warning("Slow ({$time}ms)"),
            default       => null, // Fast queries add noise without value
        };
    });
}
```

**Why map duration to log level?** Because log levels mean something — or they should. If everything is `info`, nothing stands out. A 12-second query logged as `critical` will trigger any alert rule that fires on critical log entries without writing any additional monitoring code.

**Why the config gate?** `DB::listen` fires on every single query. In a high-traffic production environment, the overhead is real. Gate it behind `app.debug` or a dedicated config flag so you control when it runs.

The 100ms warning threshold is the fastest way to surface missing indexes and N+1 problems. You see the warning in the logs, you add the index, the warning stops. No user ever had to complain.

That said — this is optional. If you add nothing else from this section, the audit trail still works. The query listener is a low-cost addition that earns its place, not a requirement.

---

## The model change logger

With a request ID threading through the system, model-level changes become meaningful entries in a traceable story rather than isolated database facts.

The implementation is a trait on any Eloquent model that needs auditing:

```php
trait ModelChangeLogger
{
    public static function bootModelChangeLogger(): void
    {
        static::updating(function ($model) {
            $changed = array_diff_assoc(
                $model->getAttributes(),
                $model->getOriginal()
            );

            // updated_at is present in every update. It is never interesting.
            unset($changed['updated_at']);

            if (empty($changed)) {
                return;
            }

            // Build a field-level diff — not just the new values,
            // but what each field changed *from*
            $diff = [];
            foreach ($changed as $key => $newValue) {
                $diff[$key] = [
                    'old' => $model->getOriginal($key) ?? 'N/A',
                    'new' => $newValue,
                ];
            }

            $model->logChanges($diff, 'updated');
        });

        static::created(function ($model) {
            $model->logChanges($model->getAttributes(), 'created');
        });

        static::deleting(function ($model) {
            // Capture the full state before deletion — after deletion,
            // getAttributes() returns nothing useful
            $model->logChanges($model->getAttributes(), 'deleted');
        });
    }

    protected function prepareLogData(array $changes, string $event): array
    {
        // Mask sensitive fields before writing — log that they changed,
        // not what they changed to
        foreach ($this->maskedAttributes ?? [] as $attr) {
            if (isset($changes[$attr])) {
                $changes[$attr] = ['old' => '[REDACTED]', 'new' => '[REDACTED]'];
            }
        }

        return [
            'event'      => $event,
            'request_id' => request()->attributes->get(RequestLogger::REQUEST_ID_ATTRIBUTE, 'N/A'),
            'time'       => now()->toDateTimeString(),
            'model_id'   => $this->getKey(),
            'changes'    => $changes,
            'user_id'    => auth()->id(),
            'ip'         => request()->ip(),
            'user_agent' => request()->userAgent(),
        ];
    }

    // Override in each model to list fields that should never appear in plain text
    protected array $maskedAttributes = [];
}
```

> **A note on console context:** When `ModelChangeLogger` runs inside a queued job or a scheduled command rather than an HTTP request, `request()->attributes->get(REQUEST_ID_ATTRIBUTE)` returns `'N/A'` — that is the intentional fallback, not a bug. If you want correlation in background jobs, generate a job-level UUID in the job constructor and inject it via `Log::shareContext()` at the start of `handle()`, the same way `RequestLogger` does for HTTP requests.

**Why `getOriginal()` rather than just the new values?** Because "the `amount` field is now 500" is half the story. "The `amount` field changed from 5000 to 500" is evidence. In a dispute, the diff proves the change happened — not just the current state.

**Why capture `deleting` before the delete, not after?**

> **Common mistake:** Using `static::deleted` instead of `static::deleting` for the deletion hook. `deleted` fires after the row is gone — `getAttributes()` returns nothing. Always use `deleting`.

`deleting` fires before the row is removed — the full record is still in memory. `deleted` fires after, and `getAttributes()` returns an empty array at that point.

---

## File-based logs and the folder segmentation problem

Here is the decision that shapes the rest of the logging architecture: where do the log entries live?

**The database option** is tempting — it is queryable, indexable, and fits naturally into a Laravel application. The problem: any table your application can write to, your application can also modify. An `UPDATE audit_logs` is valid SQL. In a regulated environment, mutable audit records are a compliance liability.

**Append-only log files** are harder to tamper with. `FILE_APPEND` at the OS level means every write goes to the end — there is no update operation, only add. Combined with filesystem permissions where the web user can write but not delete, you have logs that are genuinely difficult to alter.

```php
protected function logChanges(array $changes, string $event): void
{
    $logPath = $this->buildLogPath();

    file_put_contents(
        $logPath,
        json_encode($this->prepareLogData($changes, $event)) . PHP_EOL,
        FILE_APPEND | LOCK_EX  // LOCK_EX prevents concurrent writes corrupting the file
    );
}
```

> **Common mistake:** Using `FILE_APPEND` alone. On a busy system where multiple requests modify records simultaneously, two processes can interleave their writes and corrupt a log entry. `LOCK_EX` acquires an exclusive lock — one write completes fully before the next begins. Both flags are required.

**The folder structure problem:** if you store each record's log in a folder named after the table, you eventually have `transactions/` containing one file per transaction. A million transactions means a million files in one directory. Most filesystems handle this technically, but `ls`, backup tools, and directory enumeration operations become painfully slow.

The solution is segmentation — group records into subdirectories by ID range:

```php
protected function buildLogPath(): string
{
    $id      = $this->getKey();
    $table   = $this->getTable();

    // Segment folder: floor to the nearest 100
    // IDs 1–99 → folder "0", IDs 100–199 → folder "100", IDs 1100–1199 → folder "1100"
    $segment = (int) (floor($id / 100) * 100);

    $folder = storage_path("logs/activities/{$table}/{$segment}");

    if (!is_dir($folder)) {
        mkdir($folder, 0755, true);
    }

    $filename = strtolower(class_basename($this)) . "_{$id}.log";

    return "{$folder}/{$filename}";
}
```

The result on disk:

```
storage/logs/activities/
    transactions/
        0/
            transaction_1.log
            transaction_42.log
        1100/
            transaction_1101.log
            transaction_1102.log
        1200/
            transaction_1200.log
    users/
        0/
            user_1.log
```

Each folder contains at most 100 files. Finding a specific record's history means knowing its ID — which you always do — and reading one file. The structure scales to any number of records without any folder becoming unwieldy.

---

## Masking sensitive fields

An audit trail that contains plaintext national ID numbers or passwords creates its own compliance problem. In many jurisdictions, a log file with unredacted PII is treated like any other PII store — subject to retention limits, access controls, and deletion rights.

The approach here records that the field changed without recording what it changed to. An auditor can see that `national_id` was modified on Transaction 1101 at 14:32 by User 42 — which satisfies the audit requirement — without the log file holding the actual values.

Override `$maskedAttributes` on any model that stores sensitive data:

```php
class Transaction extends Model
{
    use ModelChangeLogger;

    protected array $maskedAttributes = [
        'national_id',
        'card_number',
    ];
}
```

---

## Where RBAC fits into this picture

Access control and audit trails are related problems. If you are logging who changed records, you should also log who *tried* to do something they were not allowed to.

Laravel's Gate provides a hook for exactly this:

```php
Gate::after(function (User $user, string $ability, bool $result) {
    if (!$result) {
        // request_id is automatically included via Log::shareContext()
        // set in RequestLogger — no need to fetch it manually here
        Log::channel('request')->warning('Authorization denied', [
            'user_id' => $user->id,
            'ability' => $ability,
            'ip'      => request()->ip(),
        ]);
    }
});
```

**Why log failures specifically?** A successful authorization attempt is normal operation. A failed attempt is a signal — a user may be probing endpoints they should not have access to, or a compromised account may be attempting privilege escalation. The pattern across multiple failed attempts becomes early warning you would not have without this hook.

One design principle worth stating: design permissions around abilities, not role names. `$user->can('transaction.approve')` expresses what the user can do, regardless of what role label they carry. `$user->role === 'admin'` breaks the moment "admin" means different things in different contexts — and in growing systems, it always eventually does.

---

## What you now have

Register `RequestLogger` as global middleware. Add `ModelChangeLogger` to any Eloquent model. Register the `Gate::after` hook.

Now: every request carries an ID. Every model change records who made it, which field changed from what value to what, when, and which request caused it. Failed permission checks are logged. Every log line across your stack shares a common correlation ID.

When a support ticket arrives saying "something changed on Transaction 1101," you open `storage/logs/activities/transactions/1100/transaction_1101.log`. You see the exact diff, the user who made it, the request ID. You grep that request ID in your request logs. You have the complete picture in under a minute.

That is the difference between an application that works and one that remembers.

---

> **The key insight from this article:** A database audit table is mutable — your application code can UPDATE it. Append-only log files with a request ID threaded through every entry give you tamper-resistant, correlated records that answer who, what, and why in one grep.

## Before you ship — checklist

- [ ] `RequestLogger` registered as global middleware in `bootstrap/app.php`
- [ ] `ModelChangeLogger` trait added to every model that handles sensitive or auditable data
- [ ] `$maskedAttributes` defined on models that store PII (passwords, national IDs, card numbers)
- [ ] `storage/logs/activities/` is writable by the web user, not deletable
- [ ] `DB::listen` is gated behind a config flag or `app.debug` — not running unconditionally in production
- [ ] `Gate::after` hook registered in `AuthServiceProvider`
- [ ] A grep for a known request ID returns results across both the request log and model change logs

---
