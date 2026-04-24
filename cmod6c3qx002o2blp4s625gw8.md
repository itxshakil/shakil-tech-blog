---
title: "Laravel Queue Architecture: Designing Background Work That Holds Up"
seoTitle: "Queue Architecture in Laravel | Jobs, Retries & Design"
seoDescription: "Learn how to design Laravel queue systems for production. Avoid common job failures, retry strategies, queue topology, and background processing relia"
datePublished: 2026-04-24T17:18:08.892Z
cuid: cmod6c3qx002o2blp4s625gw8
slug: laravel-queue-architecture-designing-background-work-that-holds-up
canonical: https://blog.shakiltech.com/wp-admin/post.php?post=1574&action=edit
cover: https://cdn.hashnode.com/uploads/covers/63bac8d8624134a0c9b53b97/b704a5f6-baec-43e1-ae29-52260cde821e.svg
tags: laravel, queue

---

**Part 2 of 4 — Laravel Architecture Patterns for Production** *~9 min read · Queue design · Job architecture · Background processing*

* * *

Background jobs are one of those features that feel solved the moment they work. You dispatch, it runs in the background, life is good. The framework handles it.

The illusion holds until scale arrives. Then a password reset email sits behind a 90-second video compression job. A job that hits a flaky external API fails, retries, fails, retries — consuming worker processes while real work waits. A bulk operation dispatches 5,000 jobs at once and the queue system staggers under the spike. A crash halfway through a file operation leaves corrupted data you do not notice for three days.

None of these are framework failures. They are design failures — decisions that were not made explicitly, so they defaulted to the wrong thing.

This article is about the decisions, and the reasoning behind them.

* * *

## What you will learn

*   Why passing an Eloquent model to a job constructor is wrong — and what to pass instead
    
*   How to design a queue topology before you are forced to by a problem
    
*   What retry strategy to use for transient, rate-limited, and permanent failures
    
*   How to write file operations that leave no corrupted state when a job crashes mid-write
    

* * *

## The mental model: a queue is not a to-do list

Most queue problems come from treating a queue as a simple list — things go in, things come out, in order. That model works fine for small volumes. It breaks under pressure.

The more accurate mental model: a queue is a **work contract** between the part of your system that knows something needs to happen and the part that will do it — separated by time, by process restarts, and by failures. The job payload is a self-contained instruction. It cannot assume anything about the context that created it still exists.

That one shift — from "list of tasks" to "self-contained work contract" — makes the rest of this article's decisions feel obvious rather than arbitrary.

* * *

## What to put in a job constructor — and what not to

The most frequent queue mistake is passing an Eloquent model to a job constructor:

```php
// This seems fine. It is not.
CompressVideo::dispatch($media);
```

When a job is dispatched, its constructor arguments are serialized into the queue payload. Laravel's `SerializesModels` trait handles this by storing the model's class and primary key — and re-fetching the model when the job runs. That sounds correct, but consider: the job might run 30 seconds later. Or three hours later if the queue is backed up. The record may have been updated, soft-deleted, or had its status changed in the time between dispatch and execution.

If the job carries a serialized model, it works with the snapshot from dispatch time. If the job carries an ID, it fetches current data at execution time.

```php
class CompressMediaVideo implements ShouldQueue
{
    public function __construct(private readonly int $mediaId) {}

    public function handle(VideoCompressorService $compressor): void
    {
        // Fetch fresh at execution time, not stale from dispatch time
        $media = Media::findOrFail($this->mediaId);

        if (!str_starts_with($media->mime_type, 'video/')) {
            return; // State may have changed since dispatch — handle gracefully
        }

        $compressor->compress($media);
    }
}
```

Pass IDs, not models. The job's responsibility is to do a unit of work starting from first principles — not to continue a conversation that started somewhere else.

* * *

## The single responsibility principle, applied to jobs

A job class should do one thing. This sounds obvious. It is regularly violated.

The problem with jobs that do multiple things: when one part fails, the whole job fails. If a job compresses a video, updates the database, sends a notification, and syncs to an external system, and the external sync times out — the compression was done, but the job retries and runs compression again. The retry was meant to retry the sync. You have wasted CPU, potentially introduced a duplicate notification, and made the job's retry behaviour unpredictable.

The cleaner design — one job per responsibility, chained. Note that the constructor is still required on each class:

```php
class CompressMediaVideo implements ShouldQueue
{
    public function __construct(private readonly int $mediaId) {}

    public function handle(VideoCompressorService $compressor): void
    {
        $media = Media::findOrFail($this->mediaId);
        $compressor->compress($media);

        // Compression done — dispatch the next unit of work independently
        UpdateMediaMetadata::dispatch($this->mediaId)->onQueue('default');
    }
}
```

Each job is independently retryable. A failure in metadata update does not re-run compression. You can add monitoring per job type and scale workers for specific job types independently.

* * *

## Queue topology: the decision nobody makes until they have to

A single `default` queue works until it does not. The moment you have both time-sensitive operations (password resets, OTPs) and long-running operations (report generation, bulk compression) on the same queue, you have a priority problem the queue itself cannot solve.

The topology decision — which queues exist and what runs on each — should be made before the first job is deployed, not when a user complains their OTP never arrived because it was stuck behind a batch export.

A practical topology:

| Queue | Purpose | Acceptable delay |
| --- | --- | --- |
| `critical` | Password resets, OTPs, security alerts | None — delay means users cannot access the system |
| `default` | Notifications, status updates, lightweight tasks | A few seconds |
| `media` | Video compression, image processing | Minutes — slow and CPU-intensive |
| `sync` | External API sync jobs | Variable — isolate third-party failures |
| `reports` | Bulk exports, large queries | Can take minutes — must never starve other queues |

Workers listen to queues in priority order:

```bash
# Critical has its own dedicated worker — nothing else competes
php artisan queue:work redis --queue=critical --timeout=30

# This worker handles default first, then media if default is empty
php artisan queue:work redis --queue=default,media --timeout=120

# Sync worker — isolated so third-party failures never affect your own queues
php artisan queue:work redis --queue=sync --timeout=60

# Reports isolated — slow timeouts do not affect other workers
php artisan queue:work redis --queue=reports --timeout=600
```

**Why isolate** `sync`**?** External API reliability is outside your control. If a third-party goes down and you are retrying aggressively, those retry jobs accumulate in the queue. On a shared queue, they compete for worker processes with your own jobs. An isolated `sync` queue means a third-party outage only affects sync workers — your application's own background work continues unaffected.

If you are on Redis and can run a dashboard, [Laravel Horizon](https://laravel.com/docs/horizon) gives you queue depth, throughput, and failed job monitoring out of the box. If you are on a database driver or a locked environment, a `php artisan queue:failed` count in a scheduled log check covers the essentials.

* * *

## Retry strategy: not all failures are equal

The default retry behaviour — try N times, then give up — is right for some jobs and wrong for others. Thinking about failure modes before writing the job saves you from the wrong defaults.

### Transient failures

Database hiccups, brief network timeouts — things that will probably succeed on retry:

```php
public int $tries  = 3;
public int $backoff = 5; // Wait 5 seconds between retries
```

### Rate-limited or service-down failures

External API unavailable, rate limit hit — back off with increasing delays:

```php
public int $tries = 5;

public function backoff(): array
{
    // 30s, 60s, 120s, 240s, 480s between retries
    return [30, 60, 120, 240, 480];
}

public function retryUntil(): \DateTime
{
    // Regardless of tries, give up after 24 hours
    // Prevents zombie jobs accumulating during prolonged outages
    return now()->addHours(24);
}
```

### Permanent failures

Malformed input, a record that will never exist — do not retry at all:

```php
public int $tries = 1;

public function failed(\Throwable $e): void
{
    // Alert immediately — this needs human attention, not retry
    logger()->critical('Permanent job failure', [
        'job'   => static::class,
        'error' => $e->getMessage(),
    ]);
}
```

**Why** `timeout` **and** `tries` **are different things, and why it matters:** A job that hits `$timeout` is killed and returns to the queue — it counts as a retry. A job that exhausts `$tries` goes to `failed_jobs`. If you set `timeout = 120` and `tries = 3` on a job that consistently takes 130 seconds, it will time out three times and land in `failed_jobs` — never completing, consuming three worker-slots in the process.

> **Common mistake:** Setting `$timeout` lower than the job's typical execution time. Every time the job runs, it gets killed before finishing, counts as a retry, and eventually exhausts `$tries` without ever succeeding. If a job needs 90 seconds, give it at least 120. The timeout is a safety net, not a target.

* * *

## Processing large backlogs: the Artisan command pattern

When you need to dispatch jobs for thousands of existing records — a backlog compression, a data migration, a bulk recalculation — the dispatch mechanism is as important as the job itself.

The pattern: an Artisan command that chunks through the database and dispatches jobs incrementally, with a `--dry-run` option before you commit to anything:

```php
class ProcessBacklogCommand extends Command
{
    protected $signature = 'media:compress-videos
        {--id=      : Process a single record by ID}
        {--dry-run  : Show what would run without dispatching anything}
        {--chunk=200: Records per batch}';

    public function handle(): void
    {
        // If --id is provided, scope the query to that single record
        $query = Media::query()
            ->where('mime_type', 'like', 'video/%')
            ->whereNull('compressed_at')
            ->when($this->option('id'), fn ($q) => $q->where('id', $this->option('id')));

        $total = $query->count();
        $this->info("Found {$total} records to process.");

        if ($this->option('dry-run')) {
            $this->info('Dry run complete — no jobs dispatched.');
            return;
        }

        $dispatched = 0;
        $query->chunkById((int) $this->option('chunk'), function ($batch) use (&$dispatched) {
            foreach ($batch as $record) {
                CompressMediaVideo::dispatch($record->id)->onQueue('media');
                $dispatched++;
            }
            $this->info("Dispatched {$dispatched} jobs so far...");
        });

        $this->info("Done. Total dispatched: {$dispatched}");
    }
}
```

**Why** `chunkById` **instead of** `chunk`**?** Laravel's `chunk()` uses SQL `OFFSET` — which means the database must count and skip N rows before returning the next batch. On a table with a million rows, the 500th batch requires skipping 100,000 rows. It gets progressively slower. `chunkById` uses `WHERE id > $lastSeen LIMIT $chunk` — a keyset cursor. The query time is constant regardless of position. For large tables, this is the difference between a command that completes in minutes and one that times out.

**Why** `--dry-run` **as a first-class option?** Because the first time you run a bulk dispatch against production data, you want to see what would happen before it happens. `--dry-run` costs nothing to add and saves numerous "I did not realise it would dispatch 40,000 jobs" moments.

* * *

## Atomic file operations: designing for crashes

Any job that writes a file must assume it will crash mid-write. At scale, it happens. The question is what state the system is in when it does.

Write directly to the target path: a crash leaves a partial, corrupted file where the original was. The damage is permanent until manual intervention.

Write to a temporary file, then rename:

```php
$inputPath = $media->full_path;
$tempPath  = $inputPath . '.tmp.' . uniqid();

// All writes go here — a crash here leaves the original untouched
$this->runFFmpeg($inputPath, $tempPath);

// Only replace if the output is actually better
if (filesize($tempPath) >= filesize($inputPath)) {
    @unlink($tempPath);
    return; // Original was already optimal — do not replace it
}

// rename() is atomic on POSIX filesystems — this either completes
// or does not happen. There is no in-between state.
rename($tempPath, $inputPath);
```

**Why compare sizes before replacing?** Re-encoding an already-compressed file can produce a larger output — the encoding overhead outweighs the savings. Always verify the output is an improvement before discarding the original.

**Why is** `rename()` **atomic?** On Linux and macOS, `rename()` is a single filesystem syscall. The kernel guarantees it either moves the file or does not — there is no half-way state where the destination file is partially written. The old file remains intact until the new one is fully in place.

* * *

## The shape of a well-designed job

Bringing it together:

```php
class CompressMediaVideo implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries   = 3;
    public int $timeout = 120;
    public int $backoff = 60;

    // Takes an ID, not a model
    public function __construct(private readonly int $mediaId) {}

    public function handle(VideoCompressorService $compressor): void
    {
        // Fetches fresh at execution time
        $media = Media::findOrFail($this->mediaId);

        // Guards for state changes since dispatch
        if (!str_starts_with($media->mime_type, 'video/')) {
            return;
        }

        $compressor->compress($media);
    }

    // Failure handling is not an afterthought
    public function failed(\Throwable $exception): void
    {
        logger()->error('Video compression failed after all retries', [
            'media_id' => $this->mediaId,
            'error'    => $exception->getMessage(),
        ]);
    }
}
```

This job takes an ID not a model, fetches fresh data at execution time, guards against state changes since dispatch, has a meaningful `failed()` handler, and has timeout, retry, and backoff values that reflect actual job characteristics — not framework defaults.

* * *

> **The key insight from this article:** Queue failures are almost never framework failures — they are design failures. Passing IDs not models, setting explicit timeout and retry values, and separating work by queue type are decisions that should be made before the first job is deployed, not after the first incident.

## Before you ship — checklist

*   \[ \] Queue topology is defined — not everything on `default`
    
*   \[ \] Every job constructor takes an ID, not an Eloquent model
    
*   \[ \] Every job has `$tries`, `$timeout`, and `$backoff` explicitly set — not left to framework defaults
    
*   \[ \] Every job has a `failed()` method that logs meaningful context
    
*   \[ \] `$timeout` is comfortably higher than the job's typical execution time
    
*   \[ \] Bulk dispatch commands use `chunkById`, not `chunk`
    
*   \[ \] Bulk commands have a `--dry-run` option
    
*   \[ \] Any job that writes a file writes to `.tmp` first, then `rename()` — never directly to the target path
    

* * *

*Previous:* [*Part 1 — The Audit Trail*](https://blog.shakiltech.com/laravel-audit-trail-building-a-system-that-remembers/)