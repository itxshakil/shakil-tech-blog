---
title: "Laravel External API Reliability: When Their System Goes Down"
seoTitle: "Laravel External API Reliability: Handling Third-Party Outag"
seoDescription: "Learn how to build Laravel applications that degrade gracefully when third-party APIs fail. Includes a reusable trait, fail-silent vs fail-loud policy per model, queued recovery, and idempotency keys — patterns from production fintech systems."
datePublished: 2026-05-04T08:51:55.097Z
cuid: cmoqynlub001h2ad36opsg9r9
slug: laravel-external-api-reliability
canonical: https://blog.shakiltech.com/laravel-external-api-reliability/
ogImage: https://cdn.hashnode.com/uploads/og-images/63bac8d8624134a0c9b53b97/3355974b-bdee-49ef-a7d4-b147ea0bd7f6.png
tags: laravel, php

---

Your code is correct. Your tests pass. Your application has been running fine for months.

Then a third-party API goes down at 2am on a Tuesday.

Registrations fail. Transaction syncs fail. Verification flows return 500s. Your queue fills with retry jobs. Users see errors they cannot act on. When the API comes back two hours later, you have a backlog of failed operations and no systematic way to recover them.

None of this was a failure of your code. It was a failure of your *design* to account for something that was always going to happen.

The question is not how to prevent third-party outages — you cannot. The question is: when they happen, does your system **behave**, or does it just **fail**?

---

## What you will build

- A trait that makes the fallback decision **explicit in the model** — not buried in a catch block
- A **fail-loud default** with a deliberate opt-in to fail-silent, per model
- A **recovery pattern** so failed syncs become retryable, not permanent data loss
- An answer to the **idempotency question** — what happens if the local write fails after the external API already succeeded?

---

## The problem with try-catch at the call site

> **New to traits?** A [PHP trait](https://www.php.net/manual/en/language.oop5.traits.php) is a reusable block of methods you can `use` inside any class. Think of it as copy-pasting methods in, except the language handles it cleanly. In Laravel, traits are commonly added to Eloquent models to share behaviour across them.

The instinctive approach to external API failures is to wrap the call in try-catch and handle it there:

```php
try {
    $externalApi->createUser($attributes);
} catch (Exception $e) {
    Log::error($e->getMessage());
    // Proceed anyway? Throw? Return false?
}
```

This is not wrong — it is incomplete. The catch block becomes the moment you are forced to decide what the system should do when the API fails, under pressure, in the middle of writing unrelated code.

That decision gets made **inconsistently** across call sites, often defaults to "log and continue" because it is easiest, and is **invisible** to anyone reading the model or service that defines the business rules.

A better approach: make the fallback behaviour explicit at the model level, not the call site.

---

## The sync trait: making resilience decisions visible

The core idea is a trait on Eloquent models that wraps API calls and exposes the failure strategy as an overridable method.

Create the file at `app/Traits/SyncsWithExternalApi.php`. Every model that calls an external API will `use` it.

> **New to Laravel's HTTP client?** Laravel ships with a wrapper around Guzzle called `Http::`. It handles timeouts, headers, retries, and response checking in a clean, readable API. [See the official docs →](https://laravel.com/docs/http-client)

```php
trait SyncsWithExternalApi
{
    /**
     * The default: fail loud. Silence requires an explicit opt-in.
     *
     * This is deliberate — the safe default is to surface failures,
     * not hide them. Models that can tolerate temporary sync failures
     * should override this method and return true.
     */
    public function shouldContinueWithoutSync(): bool
    {
        return false;
    }

    /**
     * Each model using this trait must implement createURL() and updateURL()
     * returning the external endpoint for that resource. For example:
     *
     * protected function createURL(): string
     * {
     *     return config('services.external_api.base_url') . '/users';
     * }
     */
    public static function createWithExternalApi(array $attributes): self
    {
        $url = (new static())->createURL();

        $response = Http::connectTimeout(3)  // 3s to establish the TCP connection
            ->timeout(30)                    // 30s maximum total request time
            ->withHeaders([
                'X-Requested-With' => 'XMLHttpRequest',
                'X-Forwarded-For'  => request()->ip(),
                'X-Request-ID'     => Context::get('request_id'), // see note below
            ])
            ->post($url, $attributes);

        if ($response->failed()) {
            logger()->error('External API failed on create', [
                'url'      => $url,
                'status'   => $response->status(),
                'response' => $response->json(),
            ]);

            if ((new static())->shouldContinueWithoutSync()) {
                // Explicit opt-in to fail-silent: proceed with local write only
                return static::create($attributes);
            }

            throw new Exception('External API unavailable.');
        }

        // Note: this only checks the HTTP status code, not the response body.
        // Some APIs return 200 OK with {"success": false} — add body validation
        // here based on what your specific API returns. See "What this does not solve".
        return static::create($attributes);
    }
}
```

> **Laravel version note:** `Context::get('request_id')` requires **Laravel 11** (`Illuminate\Support\Facades\Context`). On Laravel 9 or 10, replace with: `request()->attributes->get(RequestLogger::REQUEST_ID_ATTRIBUTE, '')`

---

### Why two timeout values?

```php
Http::connectTimeout(3)->timeout(30)
```

These cover two separate failure modes:

- **`connectTimeout(3)`** — aborts if the TCP handshake takes more than 3 seconds. Catches unresponsive hosts before any data is sent.
- **`timeout(30)`** — aborts if the total request takes more than 30 seconds. Catches a server that accepted the connection and then stalled.

Without either, a request to an unresponsive host may wait **75+ seconds** for the OS to give up — blocking your worker the entire time. Setting both is cheap insurance.

> **On SSL verification:** You may see `withoutVerifying()` in some codebases. This disables SSL certificate checking and is only appropriate for internal private-network services. For any public external API, leave SSL verification on. It costs nothing and catches real problems: expired certificates, man-in-the-middle attacks, DNS hijacking.

---

## Tracing failures across systems with a request ID

Notice the `X-Request-ID` header on every outgoing call. This single line is worth a brief aside.

When a sync fails and you are debugging with the third-party team, "can you look up request ID `a3f2c1...` on your end?" takes seconds. "It was a POST to your /users endpoint, around 14:30, the payload had these fields..." takes several minutes and depends on their log retention.

When both systems log the same ID, the conversation is immediate instead of archaeological.

```php
->withHeaders([
    'X-Request-ID' => Context::get('request_id'),
])
```

One header. Costs nothing in normal operation. [Correlation IDs are a standard observability pattern](https://microservices.io/patterns/observability/distributed-tracing.html) — if you are not already using them, this is a good place to start.

---

## `shouldContinueWithoutSync()` — a business decision in code

This method is the most important design element in the trait. It is not a technical setting — it encodes a business rule:

> *If this integration fails, is the primary operation still valid without it?*

The name matters. `ignoreException` sounds like you are suppressing an error. `shouldContinueWithoutSync` says exactly what it answers: can this model exist locally before the external system knows about it?

Consider two models using the same trait with opposite answers:

```php
// KYC verification — the external check IS the operation.
// If the API fails, the verification has not happened.
// Proceeding silently would mean skipping a compliance step.
class Ekyc extends Model
{
    use SyncsWithExternalApi;

    // No override — shouldContinueWithoutSync() defaults to false.
    // A failed API call is a failed operation. Full stop.
}
```

```php
// Stock transfer — the external system is a secondary record-keeper.
// Your database is the source of truth.
// A temporary outage is acceptable if you retry the sync later.
class StockTransfer extends Model
{
    use SyncsWithExternalApi;

    public function shouldContinueWithoutSync(): bool
    {
        return true; // Explicit opt-in to fail-silent
    }
}
```

Same trait. Opposite behaviour. The difference is **documented in the code** — not buried in a service class comment somewhere.

**Why default to `false`?** Fail-loud surfaces problems. Fail-silent hides them. Defaulting to silence means every model that has not thought about this will eat errors quietly. Defaulting to loud means a developer has to explicitly say "yes, I have thought about this, and silent failure is acceptable here."

---

### What a complete model looks like

The trait expects each model to implement three methods alongside the policy setting: `createURL()`, `updateURL()`, and optionally `requiredAttributes()`. Here is `StockTransfer` with all the pieces assembled:

```php
class StockTransfer extends Model
{
    use SyncsWithExternalApi;

    // Policy: can exist locally before the external system knows.
    // Failed syncs queue a retry rather than failing the operation.
    public function shouldContinueWithoutSync(): bool
    {
        return true;
    }

    // Where to create a new transfer in the external system
    protected function createURL(): string
    {
        return config('services.core_banking.url') . '/transfers';
    }

    // Where to update an existing transfer.
    // Return an array if multiple endpoints need to be notified.
    protected function updateURL(): string|array
    {
        return config('services.core_banking.url') . '/transfers/' . $this->external_id;
    }

    // Fields required in EVERY update payload — not just what changed.
    // The external system uses these to locate the record on their end.
    protected function requiredAttributes(): array
    {
        return [
            'external_id' => $this->external_id,
            'account_ref' => $this->account->external_ref,
        ];
    }
}
```

The contract the trait expects: `createURL()` and `updateURL()` say where to send data. `requiredAttributes()` says what the external system always needs to receive alongside any change. `shouldContinueWithoutSync()` says what happens when the endpoint is unreachable.

---

## Handling updates across multiple endpoints

Some models need to notify more than one external endpoint on update. The trait handles this by normalising the URL list:

```php
public function updateDetailWithExternalApi(array $data, ?bool $shouldContinueWithoutSync = null): bool
{
    $urls = $this->updateURL();

    // Accept a single string or an array — both work the same way
    if (is_string($urls)) {
        $urls = [$urls];
    }

    foreach ($urls as $url) {
        // requiredAttributes() injects fields that must always accompany updates,
        // so call sites don't have to remember to include them
        $payload = array_merge($this->requiredAttributes(), $data);

        $response = Http::connectTimeout(3)
            ->timeout(30)
            ->withHeaders([
                'X-Requested-With' => 'XMLHttpRequest',
                'X-Forwarded-For'  => request()->ip(),
                'X-Request-ID'     => Context::get('request_id'),
            ])
            ->post($url, $payload);

        if ($response->failed() || ($response->json()['success'] ?? true) === false) {
            logger()->error('External sync failed on update', [
                'url'    => $url,
                'status' => $response->status(),
                'data'   => $data,
            ]);

            $proceed = $shouldContinueWithoutSync ?? $this->shouldContinueWithoutSync();

            if ($proceed) {
                return $this->update($data);
            }

            throw new Exception("Sync failed for {$url}: " . $response->status());
        }
    }

    return $this->update($data);
}
```

**Why the `?bool $shouldContinueWithoutSync = null` parameter?** The model-level method sets the default. But callers sometimes need to override it for a specific context — a reconciliation job, for instance, might set `shouldContinueWithoutSync: true` on models that normally fail loud, because the job's purpose is to retry failures and hard-failing the whole batch over one record defeats that purpose. The parameter gives callers that flexibility without touching the model.

---

## What happens to failed syncs

The trait handles the *moment* of failure. Recovery is a separate concern — deliberately so.

When a model creates locally because the API was down, you have data your external system does not know about. Eventually this needs to reconcile. Two complementary approaches:

### Approach 1: Queued delayed retry (start here)

When a model proceeds locally because the API was down, queue a retry job immediately — but with a delay. The delay matters: if the API just went down, hitting it again in five seconds is pointless.

> **What is a job class?** In Laravel, a [job](https://laravel.com/docs/queues#creating-jobs) is a class that does background work. Create one with `php artisan make:job SyncRecordWithExternalApi`. It receives the record ID, loads the model, and calls the external API. The queue worker picks it up and runs it outside the request cycle.

Update the fail-silent branch in your trait to dispatch the retry alongside the local write:

```php
if ((new static())->shouldContinueWithoutSync()) {
    $record = static::create($attributes);

    // Do not retry immediately — the API just failed.
    // Wait 5 minutes, then attempt the sync again.
    // SyncRecordWithExternalApi is a job class you create:
    // php artisan make:job SyncRecordWithExternalApi
    SyncRecordWithExternalApi::dispatch($record->id)
        ->onQueue('sync')
        ->delay(now()->addMinutes(5));

    return $record;
}
```

> **New to queues?** Laravel's queue system defers work to a background worker process. [See the official docs →](https://laravel.com/docs/queues). For jobs that communicate with external APIs, use a dedicated queue (`.onQueue('sync')`) so a third-party outage does not back up your other work.

### Approach 2: Scheduled reconciliation (add after you've seen one outage)

A nightly job that checks for any records that were never synced — a safety net for anything that slipped through after retries were exhausted:

```php
// Runs nightly via Laravel's scheduler
$unsynced = Model::whereNull('synced_at')->cursor();
foreach ($unsynced as $record) {
    SyncRecordWithExternalApi::dispatch($record->id)->onQueue('sync');
}
```

> Add a `synced_at` column to any model where `shouldContinueWithoutSync()` returns true. Set it when the sync succeeds. That gives you an instant view of unsynced records and makes the reconciliation job trivial to write.

**Start with the delayed retry.** Add the scheduled reconciliation after you have seen your first real outage and understand your actual failure volume. Both together is more resilient than either alone.

Choosing neither — logging the error and moving on — means the inconsistency is permanent unless someone notices and manually resolves it. The cost of building the recovery path is small. The cost of skipping it compounds silently.

---

## The edge case: what if the local write fails?

Show this trait to a more experienced engineer and the first question is usually:

> *What happens if the local `create()` fails after the external API already succeeded?*

The API has the record. Your database does not. You have introduced inconsistency in the other direction.

The complete solution requires two things working together:

1. **Wrap the local write in a database transaction** so it can roll back cleanly on failure.
2. **Send an idempotency key** with the external API call, so a retry does not create a duplicate on their end.

> **What is an idempotency key?** It is a unique identifier you generate before making an API call and send as a header. If the request fails and you retry with the same key, the external API recognises it as a repeat and returns the original result — without creating a duplicate record. [Stripe's idempotency docs](https://stripe.com/docs/api/idempotent_requests) are a good reference for how this works in practice.

```php
public static function createWithExternalApi(array $attributes): self
{
    $url = (new static())->createURL();

    // Generate the key before the call.
    // On retry, send the same key — the external API returns
    // the same result without creating a duplicate.
    $idempotencyKey = Str::uuid()->toString();

    $response = Http::connectTimeout(3)
        ->timeout(30)
        ->withHeaders([
            'X-Idempotency-Key' => $idempotencyKey,
            'X-Request-ID'      => Context::get('request_id'),
        ])->post($url, $attributes);

    if ($response->failed()) {
        // API failed before writing — safe to retry with the same key
        throw new Exception('External API unavailable.');
    }

    // API succeeded. Wrap the local write in a transaction so it can
    // roll back cleanly if something goes wrong (e.g. a validation error
    // or a database constraint). If the transaction fails, retrying the
    // whole operation with the same idempotency key is safe — the API
    // won't create a duplicate on the next attempt.
    return DB::transaction(function () use ($attributes) {
        return static::create($attributes);
    });
}
```

> **Check your API's documentation.** Not all external APIs support idempotency keys — it depends on the provider. When they are available, use them. Without them, the only safe option after a partial failure is manual reconciliation, and manual steps under incident pressure go wrong.

---

## What this does not solve

A pattern is a floor, not a ceiling. Be clear about its limits:

- **200 responses with error payloads.** `$response->failed()` checks the HTTP status code. Some APIs return `200 OK` with `{"success": false}` in the body. The update method above checks for this, but the create method does not — add body validation based on what your specific API returns.
- **Silent data corruption.** A successful response does not guarantee the data was stored correctly. For critical operations, validate the response body against what you sent.
- **Partial writes on the external end.** Some APIs accept a request, return success, and write only part of the data. Only end-to-end testing and periodic reconciliation catches this.
- **Providers that don't support idempotency keys.** Nothing in this pattern solves the duplicate-create problem if the external API does not support them.

---

## Where you end up

If you apply this consistently, one thing changes: the fallback decision is not made under pressure at 2am. It was made when the model was written, by someone who understood the business rule, and it is readable in the code by anyone who comes after them.

`shouldContinueWithoutSync()` on each model. Logged context on every failure. A recovery path built in, not improvised.

The 2am outage will still happen. This just means you will have a plan when it does.

---

## Before you ship — checklist

- [ ] Every `Http::` call has both `connectTimeout()` and `timeout()` set explicitly
- [ ] `withoutVerifying()` is not present on any call to a public external API
- [ ] `shouldContinueWithoutSync()` return value is a **conscious decision** per model — not left as the default by accident
- [ ] Models where `shouldContinueWithoutSync()` is `true` have a `synced_at` column and a recovery path (queued retry, scheduled reconciliation, or both)
- [ ] Every failed sync logs: URL, status code, response body, and the payload sent
- [ ] `X-Request-ID` is passed as a header on all outgoing API calls
- [ ] `requiredAttributes()` is defined on any model that must include identifying fields in every update
- [ ] Idempotency keys are used for any external API that supports them

---

> **The key insight:** The fallback decision — fail loud or continue without sync — is a business rule, not a technical setting. Encoding it as a named method on the model (`shouldContinueWithoutSync()`) makes the decision readable in the code, consistent across every call site, and impossible to forget to make.

---

*Previous: [Part 3 — Secure File Uploads](./part-3-secure-file-uploads.md)*

---

## What this series built

**Part 1** turned "we don't know what changed" into a traceable system: field-level diffs, append-only logs, request IDs that correlate across every log channel, and permission failures visible before they become incidents.

**Part 2** moved from "dispatch and hope" to intentional queue design: topology that separates work by characteristics, job constructors that carry IDs not snapshots, retry strategies matched to actual failure modes.

**Part 3** replaced a single file-type check with seven independent ones — each named, each closing a specific attack vector.

**Part 4** made "what happens when their API goes down?" answerable before it becomes an incident.

The common thread: decisions that are usually made implicitly — or not at all until something breaks — made explicit in code, with the reasoning attached.

---

## Further reading

- [Laravel HTTP Client](https://laravel.com/docs/http-client) — official docs for `Http::`, including retry helpers and middleware
- [Laravel Queues](https://laravel.com/docs/queues) — how to set up workers, delayed dispatch, and failure handling
- [Laravel Job Classes](https://laravel.com/docs/queues#creating-jobs) — how to create the `SyncRecordWithExternalApi` job used in the recovery pattern
- [Stripe: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests) — the clearest real-world explanation of idempotency keys
- [Microservices: Distributed Tracing pattern](https://microservices.io/patterns/observability/distributed-tracing.html) — why correlation IDs matter across systems
- [Honeybadger for PHP](https://www.honeybadger.io/for/php/) — exception monitoring that surfaces failed syncs before users report them