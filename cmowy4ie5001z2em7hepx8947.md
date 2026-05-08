---
title: "How We Implemented Content Security Policy (CSP) in Our Laravel App "
seoDescription: "A real-world guide to adding Content Security Policy to Laravel — nonce generation, Vite integration, violation reporting, and deploying without break"
datePublished: 2026-05-08T13:23:41.220Z
cuid: cmowy4ie5001z2em7hepx8947
slug: how-we-implemented-content-security-policy-csp-in-our-laravel-app
canonical: https://blog.shakiltech.com/how-we-implemented-content-security-policy-csp-laravel/
cover: https://cdn.hashnode.com/uploads/covers/63bac8d8624134a0c9b53b97/1408c1ad-6824-4fc5-b824-a20ba1b67243.png
tags: laravel, security, php, csp

---

# How We Implemented Content Security Policy (CSP) in Our Laravel App 🔒

*Tags: Laravel, Security, CSP, Middleware, PHP*

* * *

Our pentest report had one line that stopped us cold:

> *"Application does not implement Content-Security-Policy headers. XSS payloads executed without restriction."*

We had Sanctum, CSRF tokens, input validation — all the standard Laravel security checklist items. But we had no CSP. And without it, a single successful XSS attack could exfiltrate session cookies, inject malicious scripts, or silently redirect users to attacker-controlled pages — all from our own domain.

This is the story of how we added CSP to a production Laravel application without breaking anything, how we built a violation reporting pipeline, and the things we wish we'd known before starting.

* * *

**What's in this guide**

This post is written to be useful regardless of where you're starting from:

*   **Never heard of CSP?** Start from the top. The first two sections give you the mental model before any code appears.
    
*   **Know what CSP is but haven't implemented it?** Jump to [The Implementation](#the-implementation).
    
*   **Already running CSP and want to tighten it or add reporting?** Jump to [CSP Violation Reporting](#csp-violation-reporting) or the [Pre-Enforcement Checklist](#pre-enforcement-checklist).
    

* * *

## What Content Security Policy Actually Does

When a browser loads a page, it executes whatever scripts, styles, fonts, and images are on it — regardless of where they came from. That's what makes XSS dangerous: if an attacker manages to inject a `<script>` tag into your HTML, the browser runs it without question.

CSP is an HTTP response header that changes this. It tells the browser: *"Only trust resources from these specific sources. Anything else — block it."*

```plaintext
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'; object-src 'none'
```

With that header in place, even if an attacker injects a `<script>` tag, the browser refuses to execute it. The tag has no valid **nonce** — a cryptographically random token generated fresh for each request. Scripts carrying the matching nonce run. Everything else is blocked.

> **New to this?** MDN has the [definitive CSP reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) if you want to go deeper on any specific directive.

* * *

## What Is a Nonce?

You'll see the word "nonce" throughout this post, so let's settle it quickly before writing any code.

A **nonce** (short for *number used once*) is a random string generated fresh on every single page load. The server puts it in the CSP header, and also stamps it onto every `<script>` or `<style>` block it trusts:

```plaintext
// In the HTTP header the server sends:
Content-Security-Policy: script-src 'nonce-k8Hv2mXpQ3'

// On the script tag in your HTML:
<script nonce="k8Hv2mXpQ3">...</script>
```

The browser sees both, checks they match, and runs the script. If an attacker injects a `<script>` tag, it has no nonce — or the wrong one — so it gets blocked. Since the nonce changes on every request, an attacker who somehow saw yesterday's nonce can't reuse it today.

**Why not just use a static secret key?** Because a static value could be cached, leaked in logs, or extracted from your source code. A nonce only needs to survive one request. After that, it's worthless.

**What a nonce is not:**

*   It is not a replacement for input sanitisation. A nonce stops injected scripts from *running*. It doesn't stop them from being *stored* in your database.
    
*   It is not encryption. Anyone who can see the page source can see the nonce — that's fine, because it expires the moment the request ends.
    
*   It is not a CSRF token. They look similar in concept but solve completely different problems.
    

That's it. When you see `nonce="{{ $cspNonce }}"` in templates below, it simply means: *"the server trusts this specific block."*

* * *

## The Naive Approach (and Why It Fails)

Most teams discover CSP when a security audit flags it, then reach for the quickest fix: a static header in `nginx.conf` or `apache.conf`.

This works for about five minutes. Then the inline JavaScript breaks. Alpine.js stops. Livewire stops. Any `<script>` block that doesn't come from a separate file gets blocked. The knee-jerk reaction is:

```plaintext
script-src 'self' 'unsafe-inline'
```

Done. Everything works again. And CSP is now completely useless — `'unsafe-inline'` means *any* inline script can run, including one an attacker injected.

The correct approach is a **nonce-based CSP generated per request inside Laravel**, where every trusted inline script carries a token only the server knows. That's what we built.

* * *

## The Mindset Shift: Write CSP-Friendly Code

Before writing a single line of middleware, you need to change how you write templates. CSP doesn't just enforce a header — it enforces discipline about where your code lives.

**Move JavaScript out of your HTML.** Event handler attributes are the main offender:

```html
<!-- ❌ This is blocked by any sensible CSP -->
<button onclick="submitForm()">Submit</button>
<script>function submitForm() { ... }</script>

<!-- ✅ This is CSP-friendly — JS lives in an external file -->
<button id="submit-btn">Submit</button>
<script src="/js/form.js"></script>
```

**Move styles out of your HTML too.** The `style=""` attribute on elements cannot carry a nonce — there's no mechanism to whitelist individual instances. Your only options with a strict policy are "allow all of them" or "block all of them." Build the habit of using CSS classes from the start.

**For inline code you genuinely can't move** — a config value JavaScript needs, critical above-the-fold CSS — the nonce is your escape hatch:

```html
<script nonce="{{ $cspNonce }}">
    window.APP_ENV = "{{ config('app.env') }}";
</script>
```

A nonce is generated fresh on every request. An attacker can't predict it. A script they inject has no nonce, so it gets blocked even if everything else fails.

> **Pro Tip:** Never hardcode a nonce, never reuse one. Generate it with `random_bytes()` — not `uniqid()`, not `md5(time())`.

* * *

> **Just want the output without the reading?** I built a free [Laravel CSP Generator](https://blog.shakiltech.com/tools/laravel-csp-generator) — toggle your CDNs and integrations, and it generates the full policy string, the PHP middleware method, and an adaptive pre-enforcement checklist live. Come back here for the why behind each decision.

* * *

## The Implementation

We put all security headers in one `SecureHeadersMiddleware` rather than spreading logic across `nginx.conf`, `.htaccess`, and multiple middleware files.

### Step 1: Generate the Nonce and Share It

```bash
php artisan make:middleware SecureHeadersMiddleware
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Illuminate\Support\Facades\View;
use Symfony\Component\HttpFoundation\Response;

class SecureHeadersMiddleware
{
    // Disable in local/testing to avoid log noise from developer browsers
    // full of extensions. Automatically driven by the current environment.
    private bool $allowReporting;

    public function __construct()
    {
        $this->allowReporting = !app()->environment(['local', 'testing']);
    }

    public function handle(Request $request, Closure $next): Response
    {
        $nonce = base64_encode(random_bytes(16));

        // Share with all Blade templates automatically — no manual passing needed
        View::share('cspNonce', $nonce);

        // Tell Vite to inject this nonce on its bootstrapping inline script.
        // Skip this and @vite() silently breaks under a strict CSP.
        Vite::useCspNonce($nonce);

        /** @var Response $response */
        $response = $next($request);

        $this->addContentSecurityPolicy($response, $nonce, $request);
        $this->addClickjackingProtection($response);
        $this->addStrictTransportSecurity($response, $request);
        $this->addMiscSecurityHeaders($response);
        $this->removeUnwantedHeaders($response);

        if ($this->allowReporting && $this->supportsReportTo($request)) {
            $this->addCspReportingEndpoint($response);
        }

        return $response;
    }
}
```

Two ordering rules matter here. The nonce must be generated *before* `$next($request)` because the view renders during that call. The headers are set *after* because they need the response object. Swap either and things break silently.

### Step 2: Define the CSP Policy

```php
protected function addContentSecurityPolicy(Response $response, string $nonce, Request $request): void
{
    $isLocal = app()->environment('local');

    $policy = [
        "default-src 'self'",
        "script-src 'self' 'nonce-{$nonce}' https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://embed.tawk.to",
        "style-src 'self' 'nonce-{$nonce}' https://fonts.googleapis.com https://cdnjs.cloudflare.com",
        "style-src-elem 'self' 'nonce-{$nonce}' https://fonts.googleapis.com https://cdnjs.cloudflare.com",
        "style-src-attr 'none'",
        "font-src 'self' data: https://fonts.gstatic.com https://cdnjs.cloudflare.com",
        "img-src 'self' data: https://cdn.jsdelivr.net https://cdnjs.cloudflare.com https://your-cdn.example.com",
        "frame-src 'self' https://your-video-host.example.com",

        // Vite's HMR uses a WebSocket on localhost — only needed in local dev.
        // Never ship localhost entries to production.
        "connect-src 'self'" . ($isLocal ? " ws://localhost:5173 wss://localhost:5173" : ""),

        "object-src 'none'",
        "base-uri 'self'",
        "frame-ancestors 'none'",
    ];

    // upgrade-insecure-requests on a local HTTP server triggers confusing
    // mixed-content errors that have nothing to do with your code.
    if (!$isLocal) {
        $policy[] = 'upgrade-insecure-requests';
    }

    if ($this->allowReporting) {
        $policy[] = 'report-uri /api/csp-violation-report';
    }

    $response->headers->set('Content-Security-Policy', implode('; ', $policy));
}
```

**Directive reference — what each one actually does:**

| Directive | What it controls | Our value |
| --- | --- | --- |
| `default-src` | Fallback for any type not explicitly listed | `'self'` |
| `script-src` | JavaScript execution | `'self'` + nonce + CDN allowlist |
| `style-src` | CSS `@import` rules within stylesheets; base fallback | `'self'` + nonce + CDN allowlist |
| `style-src-elem` | `<link rel="stylesheet">` and `<style>` blocks | `'self'` + nonce + CDN allowlist |
| `style-src-attr` | Inline `style=""` attributes | `'none'` (blocked entirely) |
| `font-src` | Font files | `'self'` + data URIs + Google Fonts |
| `img-src` | Images | `'self'` + data URIs + CDN |
| `frame-src` | `<iframe>` sources | `'self'` + trusted video host |
| `connect-src` | `fetch()`, XHR, WebSockets, SSE | `'self'` + localhost WS in dev |
| `object-src` | Browser plugins (Flash, Java) | `'none'` |
| `base-uri` | The `<base>` tag's `href` | `'self'` |
| `frame-ancestors` | Who can embed this page in an iframe | `'none'` |

`default-src 'self'` means anything not covered by a specific directive falls back to same-origin only.

`script-src` is where most of the work happens. The nonce is per-request, cryptographically random, and unpredictable. In CSP Level 3, the presence of a nonce automatically overrides `'unsafe-inline'` for supporting browsers — so even if you add it as a temporary fallback, modern browsers ignore it in favour of nonce validation.

`connect-src` controls everything your JavaScript can talk to: `fetch()`, `XMLHttpRequest`, WebSockets, and Server-Sent Events. Vite's Hot Module Replacement uses a WebSocket on `localhost:5173` to push changes to your browser during development. Without it in `connect-src`, HMR fails silently and you're doing hard refreshes all day. In production, that entry must not appear — hence the environment check.

`object-src 'none'` blocks Flash and all other browser plugins. There's no reason to allow these in 2026.

`base-uri 'self'` prevents an attacker from injecting `<base href="https://evil.com">`, which would silently redirect all relative URLs — links, form actions, script paths — to their server.

`frame-ancestors 'none'` is clickjacking protection at the CSP level: your pages cannot be embedded in iframes on other domains. Stronger than `X-Frame-Options` for modern browsers; we keep `X-Frame-Options` too for legacy ones.

`upgrade-insecure-requests` tells the browser to automatically upgrade HTTP sub-resource requests to HTTPS. We skip it in local development because it causes confusing mixed-content errors on a plain HTTP dev server.

#### Deep Dive: `style-src`, `style-src-elem`, and `style-src-attr`

Most developers assume these three are aliases for the same thing and only set `style-src`. They're not — and that misunderstanding is what causes hours of debugging when styles randomly break.

**The one-line summary:** `style-src` is the fallback. `style-src-elem` controls `<style>` blocks and `<link>` tags. `style-src-attr` controls `style=""` attributes on elements. Set all three explicitly, or the browser decides how to inherit, which is rarely what you want.

Here's the breakdown:

`style-src` is the base rule that covers everything CSS-related when the more specific directives aren't set. Even when you do set the others, `style-src` still governs one thing they don't: **CSS** `@import` **rules inside your own stylesheets**. `@import url('https://fonts.googleapis.com/...')` is controlled by `style-src`, not `style-src-elem`.

`style-src-elem` controls two things: `<link rel="stylesheet">` tags and `<style>` blocks. The important detail is that `<style>` blocks *can* carry a nonce, so you can precisely whitelist only the inline styles you wrote:

```html
{{-- Trusted — nonce matches the header --}}
<style nonce="{{ $cspNonce }}">
    .hero { background: url('/img/hero.jpg') center/cover; }
</style>

{{-- Blocked — no nonce --}}
<style>.injected { color: red; }</style>
```

External stylesheets via `<link href="...">` don't need a nonce — they're controlled by the domain allowlist. Nonces only apply to inline content.

`style-src-attr` controls only the inline `style=""` attribute on HTML elements — and this is the problem child. `style=""` attributes **cannot carry a nonce**. There is no way to whitelist specific ones. Your only two options are `'unsafe-inline'` (trusts all of them, including anything injected) or `'none'` (blocks all of them). We use `'none'`.

```plaintext
style-src       → CSS @import in files + base fallback  → 'self' + nonce + CDN
style-src-elem  → <link> tags + <style> blocks          → nonce-gated
style-src-attr  → style="" attributes                   → 'none', blocked entirely
```

**Before you enable this:** audit your JavaScript-driven UI libraries. Drag-and-drop, tooltips, and modals commonly set `style="top: 48px; left: 200px"` dynamically — every one of them will break under `style-src-attr 'none'`. Run in report-only mode first and your logs will tell you exactly which libraries are the offenders before anything breaks in production.

* * *

### Step 3: Use the Nonce in Blade

```html
{{-- Vite assets — no nonce attribute needed on the tag itself.
     Vite::useCspNonce() in the middleware handles the inline bootstrapper. --}}
@vite(['resources/css/app.css', 'resources/js/app.js'])

{{-- Inline script blocks you write manually --}}
<script nonce="{{ $cspNonce }}">
    const config = @json($appConfig);
</script>

{{-- Inline style blocks --}}
<style nonce="{{ $cspNonce }}">
    .hero { background: url('/img/hero.jpg') center/cover; }
</style>

{{-- Livewire v2 --}}
@livewireScripts(['nonce' => $cspNonce])

{{-- Livewire v3 --}}
@livewireScriptConfig(['nonce' => $cspNonce])
```

**One thing to be clear on:** `<script src="...">` and `<link rel="stylesheet" href="...">` tags pointing to external files do **not** need a nonce attribute. The browser permits them if their domain is on the allowlist. Nonces exist purely for inline content that has no domain to verify against.

### Step 4: Add the Rest of the Security Header Stack

CSP is one layer. The same middleware handles the remaining browser security headers — each in a focused method:

```php
protected function addClickjackingProtection(Response $response): void
{
    // Legacy browsers fall back to this; modern ones use frame-ancestors from CSP
    $response->headers->set('X-Frame-Options', 'DENY');
}

protected function addStrictTransportSecurity(Response $response, Request $request): void
{
    // Check X-Forwarded-Proto too — without this, HSTS is never sent when
    // you're behind a load balancer that terminates SSL before Laravel sees the request
    $isHttps = $request->isSecure()
        || strcasecmp((string) $request->header('X-Forwarded-Proto', ''), 'https') === 0;

    if (!$isHttps) {
        return;
    }

    $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload');
}

protected function addMiscSecurityHeaders(Response $response): void
{
    // Prevents browsers from MIME-sniffing a response away from the declared content-type
    $response->headers->set('X-Content-Type-Options', 'nosniff');

    // Controls what's sent in the Referer header on navigation
    $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');

    // Deprecated and ignored by modern browsers, but harmless to include
    $response->headers->set('X-XSS-Protection', '1; mode=block');

    // Restrict access to sensitive browser APIs
    $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=(self), camera=(self), payment=(), fullscreen=*');
}

protected function removeUnwantedHeaders(Response $response): void
{
    // Both calls are needed: Symfony's header management and PHP's header() function
    // operate at different levels. Without both, X-Powered-By: PHP/8.x can still leak.
    foreach (['X-Powered-By', 'Server'] as $header) {
        $response->headers->remove($header);
        @header_remove($header);
    }
}
```

### Step 5: Register the Middleware

```php
// Laravel 11 — bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        \App\Http\Middleware\SecureHeadersMiddleware::class,
    ]);
})

// Laravel 10 — app/Http/Kernel.php
protected $middlewareGroups = [
    'web' => [
        // ...
        \App\Http\Middleware\SecureHeadersMiddleware::class,
    ],
];
```

Attaching it to the `web` group means your JSON API routes — which don't render Blade views — don't receive an unnecessary CSP header. For high-throughput APIs that's a meaningful separation.

* * *

## CSP Violation Reporting — Your Early Warning System 🚨

Running CSP without a reporting endpoint is like setting a burglar alarm with no monitoring. You need to know when the policy blocks something — to catch misconfigurations before users do, and to detect real injection attempts.

### What a Violation Report Looks Like

When a browser blocks something, it POSTs a JSON payload to your report endpoint. Here's what that looks like:

```json
{
  "csp-report": {
    "document-uri": "https://yourapp.com/dashboard",
    "referrer": "",
    "violated-directive": "script-src-elem",
    "effective-directive": "script-src-elem",
    "original-policy": "default-src 'self'; script-src 'self' 'nonce-abc123'",
    "blocked-uri": "https://evil.com/injected.js",
    "status-code": 200
  }
}
```

The `blocked-uri` tells you what was blocked. The `violated-directive` tells you which rule caught it. Together they tell you immediately whether this is a legitimate resource you forgot to allowlist or something that should never have been there.

> **Hosted alternative:** If you'd rather not build your own reporting pipeline, [report-uri.com](https://report-uri.com) by Scott Helme is a dedicated CSP report collection and analysis service. It handles noise filtering, dashboards, and alerting for you.

### The Route

```php
// routes/api.php
Route::post('/csp-violation-report', [CspReportController::class, 'store'])
    ->middleware('throttle:5');
```

`throttle:5` limits to 5 reports per minute per IP. Without rate limiting, an attacker can flood your logging infrastructure by triggering violations in a loop — exhausting disk space or burying real threats in noise.

### The Controller

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\RateLimiter;

class CspReportController
{
    public function store(Request $request)
    {
        $body = json_decode($request->getContent(), true);
        $report = $body['csp-report'] ?? $body ?? [];

        if (empty($report) || !is_array($report)) {
            return response()->noContent();
        }

        $json = json_encode($report);
        $ip = $request->ip();
        $blockedUri = data_get($report, 'blocked-uri', '');
        $violatedDirective = data_get($report, 'violated-directive', '');

        // Browser extensions are the biggest source of CSP noise.
        // Ad blockers, password managers, and DevTools extensions all fire reports.
        // They're not actionable — filter them immediately.
        if (
            str_contains($json, 'chrome-extension://') ||
            str_contains($json, 'moz-extension://') ||
            str_contains($json, 'safari-extension://')
        ) {
            return response()->noContent();
        }

        // Low-risk: usually your own code or a library doing something non-standard.
        // Review weekly, not immediately.
        $lowRiskUris = ['about:blank', 'blob:', 'data:', 'inline', 'eval'];

        if (in_array($blockedUri, $lowRiskUris)) {
            Log::channel('csp_low')->info('Low-risk CSP violation', compact('ip', 'report'));
            return response()->noContent();
        }

        // High-risk: an external domain, an unknown script, potentially injected content.
        // Log with full context. Alert if sustained.
        Log::channel('csp_high')->warning('High-risk CSP violation', compact('ip', 'report'));

        $this->checkForThresholdAlert($ip, $blockedUri, $violatedDirective);

        return response()->noContent();
    }

    protected function checkForThresholdAlert(string $ip, string $uri, string $directive): void
    {
        // Same IP, same directive, 10+ times in 60 seconds = probe, not noise.
        $key = "csp_alert:$ip:{$directive}:$uri";

        if (RateLimiter::tooManyAttempts($key, 10)) {
            Log::alert('CSP threshold exceeded — possible active attack', [
                'ip'        => $ip,
                'uri'       => $uri,
                'directive' => $directive,
            ]);
            return;
        }

        RateLimiter::hit($key, 60);
    }
}
```

### Log Channels

```php
// config/logging.php
'csp_low' => [
    'driver' => 'single',
    'path'   => storage_path('logs/csp-low-risk.log'),
    'level'  => 'info',
],

'csp_high' => [
    'driver' => 'single',
    'path'   => storage_path('logs/csp-high-risk.log'),
    'level'  => 'warning',
],
```

Separate files let you set different retention policies — low-risk logs rotate weekly, high-risk logs stay for 90 days. In production, wire `Log::alert()` into Slack or PagerDuty so a spike in high-risk violations at 2 AM wakes someone up.

### The Modern Report-To Header

For Chrome 70+, Edge 79+, and Firefox 60+, the [Reporting API](https://developer.mozilla.org/en-US/docs/Web/API/Reporting_API) (`Report-To` header) is more efficient than `report-uri` — reports are batched, and the endpoint is cached so the browser sends reports even from pages that didn't include the header.

```php
protected function supportsReportTo(Request $request): bool
{
    $ua = $request->header('User-Agent', '');

    return (bool) preg_match(
        '/(Chrome\/([7-9][0-9]|[1-9][0-9]{2,}))|(Firefox\/(6[0-9]|[7-9][0-9]|[1-9][0-9]{2,}))|(Edg\/([7-9][0-9]|[1-9][0-9]{2,}))|(Version\/(1[3-9]|[2-9][0-9])) Safari\//',
        $ua
    );
}

protected function addCspReportingEndpoint(Response $response): void
{
    $reportTo = [
        'group'              => 'csp-endpoint',
        'max_age'            => 10886400, // 126 days — browser caches this endpoint
        'endpoints'          => [
            ['url' => url('/api/csp-violation-report')],
        ],
        'include_subdomains' => true,
    ];

    $response->headers->set('Report-To', json_encode($reportTo, JSON_UNESCAPED_SLASHES));
}
```

We keep `report-uri` in the CSP policy as a fallback for browsers that don't support the Reporting API. Both can coexist.

* * *

## Pre-Enforcement Checklist

Run through this before switching from report-only to enforcement mode:

*   \[ \] `Vite::useCspNonce($nonce)` is called *before* `$next($request)` in the middleware
    
*   \[ \] `@vite()`, `@livewireScripts()`, and `@livewireScriptConfig()` all pass the nonce
    
*   \[ \] Every hand-written inline `<script>` block has `nonce="{{ $cspNonce }}"`
    
*   \[ \] Every hand-written inline `<style>` block has `nonce="{{ $cspNonce }}"`
    
*   \[ \] `onclick=""`, `onsubmit=""`, and other event handler attributes are removed from templates
    
*   \[ \] `style=""` attributes are replaced with CSS classes (or the library using them is documented)
    
*   \[ \] All CDN domains you load from are in the appropriate allowlist
    
*   \[ \] `connect-src` includes your API endpoints, WebSocket hosts, and analytics targets
    
*   \[ \] `connect-src` does **not** include `localhost` in your production policy
    
*   \[ \] The violation report endpoint is live, throttled, and log channels are configured
    
*   \[ \] Report-only mode has run in production for at least one week with no new high-risk violations
    

* * *

## Deploy Safely: Start in Report-Only Mode

If you're adding CSP to an existing app, don't jump straight to enforcement. Swap the header name first:

```php
// Change this:
$response->headers->set('Content-Security-Policy', implode('; ', $policy));

// To this, temporarily:
$response->headers->set('Content-Security-Policy-Report-Only', implode('; ', $policy));
```

The browser reports violations but blocks nothing. Run it in production for one to two weeks and watch `csp-high-risk.log`. Every entry is either a resource to allowlist or evidence of something that should never have been there. Once the log goes quiet, flip back to `Content-Security-Policy`.

This step is what separates a smooth rollout from an angry support queue.

> **Verify your headers:** Use [securityheaders.com](https://securityheaders.com) to grade your response headers. Use Google's [CSP Evaluator](https://csp-evaluator.withgoogle.com/) to check your policy for common weaknesses. Both are free.

* * *

## What We'd Do Differently

**Drop** `X-XSS-Protection`**.** It's deprecated and ignored by modern browsers. Older IE versions had a bug where it could be exploited. We kept it because it doesn't actively hurt anything, but new implementations shouldn't bother.

**Separate concerns if your team is large.** Bundling all headers in one middleware works well for us, but a dedicated `CspMiddleware` is easier to unit test and simpler to replace later. [Spatie's laravel-csp](https://github.com/spatie/laravel-csp) package gives you a structured, per-policy-class approach with first-class test support if you want to skip the custom build entirely.

**Use hashes for truly static inline content.** If you have a small piece of inline CSS that can't move to a stylesheet — a background colour from a database value, for example — CSP supports SHA-256 hashes as an alternative to nonces. You precompute the hash of the exact string and whitelist it. Surgical precision with no runtime overhead.

**Explicitly exclude the** `api` **group from the middleware.** Our implementation attaches the middleware to the `web` group, which already keeps it off pure API routes. If your app registers any routes globally — or if you ever move to a flat route file — API responses will silently start receiving a CSP header they don't need. Being explicit with a group exclusion is safer than relying on registration order:

```php
// Laravel 11 — bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        \App\Http\Middleware\SecureHeadersMiddleware::class,
    ]);
    // api group intentionally excluded — JSON responses don't need CSP
})
```

* * *

## The Result

After two weeks in report-only mode and one afternoon of allowlist tuning, we flipped to enforcement. Our security headers score went from D to A+ on [securityheaders.com](https://securityheaders.com).

Since then, `csp-high-risk.log` has caught two incidents where a third-party CDN attempted to inject tracking scripts we never authorised. The middleware caught what code review wouldn't have.

If you want to configure your own policy without writing it from scratch, the [Laravel CSP Generator](https://csp-generator.shakiltech.com lets you do it visually — toggle your CDNs, integrations, and environment settings, and it outputs the full policy string and PHP middleware method ready to paste. The pre-enforcement checklist is built in and adapts to whatever you've configured.

CSP isn't a silver bullet — it's one layer in a defence-in-depth approach alongside CSRF protection, input validation, prepared statements, and HTTPS. But unlike most defences, it actively stops an entire class of attacks at the browser level, rather than hoping your application-layer defences caught everything upstream.

Start in report-only mode today. Watch your logs for a week. Then enforce. 🚀

* * *

## Get the Quick Reference PDF

If you want a single document to keep open while you're building — the full directive reference, the pre-enforcement checklist, common mistakes, and quick-start snippets — I put it all in a 4-page PDF.

Drop your email below and it'll land in your inbox immediately. No series, no spam — just the PDF.

> [**→ Get the Laravel CSP Quick Reference PDF**](https://csp-generator.shakiltech.com#guide)
> 
> 4 pages · All 13 directives · Pre-enforcement checklist · 5 common mistakes · Free

* * *

## Frequently Asked Questions

**Does CSP break existing Laravel apps?**

It can — specifically if you have inline scripts or styles without nonces. Use `Content-Security-Policy-Report-Only` first to surface everything that would be blocked, fix your templates, then switch to enforcement. This is the only safe migration path for an app that's already in production.

**Do I need CSP if I already have CSRF protection?**

Yes. They protect against different attacks. CSRF prevents forged requests originating from other domains. CSP prevents injected scripts from *running on your domain*. One does not substitute for the other.

**Do I need CSP if my app has no user-generated content?**

Yes. XSS doesn't require users to submit anything. A vulnerable dependency, a compromised CDN script, or a DOM-based XSS through a URL parameter are all real attack vectors that have nothing to do with user uploads.

**My Vite HMR stopped working after I added CSP. What's wrong?**

Two things to check. First, confirm `Vite::useCspNonce($nonce)` is called before `$next($request)`. Second, confirm `connect-src` includes `ws://localhost:5173` (or your Vite port) in your local environment. HMR uses a WebSocket and fails silently if that WebSocket connection is blocked.

**What's the difference between** `report-uri` **and** `Report-To`**?**

`report-uri` is the original mechanism and is widely supported. `Report-To` is the newer Reporting API — reports are batched, the endpoint URL is cached by the browser for months, and it covers more than just CSP. Use both: `report-uri` as the universal fallback, `Report-To` for modern browsers. Alternatively, skip rolling your own and use [report-uri.com](https://report-uri.com).

**How do I debug CSP violations locally?**

Open DevTools → Console. Every CSP violation logs the exact blocked resource and the violated directive. That's your fastest debugging tool. If you're in report-only mode, the same violations also appear without anything actually breaking.

**How do I handle CSP with Livewire?**

Livewire injects inline JavaScript that needs the nonce. For v2: `@livewireScripts(['nonce' => $cspNonce])`. For v3: `@livewireScriptConfig(['nonce' => $cspNonce])`. Both are shown in Step 3 above.

**What about Google Analytics or Tag Manager?**

Add `https://www.googletagmanager.com` to `script-src` and `https://www.google-analytics.com` to both `script-src` and `connect-src`. GTM's custom HTML tags require `'unsafe-inline'`, which weakens your policy — server-side GTM or direct GA4 integration are cleaner alternatives worth the migration cost.

**Can I use Spatie's** `laravel-csp` **package instead of a custom middleware?**

Absolutely. [Spatie's package](https://github.com/spatie/laravel-csp) provides a structured, per-policy-class approach with built-in nonce support and first-class test utilities. Our custom middleware made sense for us because we wanted every security header in one place without a package dependency. Either approach is valid — choose based on your team's preference for control versus convention.

* * *

*Have you implemented CSP on a production Laravel app? Whether you had a smooth rollout, hit a rough edge with a third-party library, or are still sitting in report-only mode wondering what to do next — drop a comment. I'd love to hear where you got stuck, or what blocked resources surprised you most when you first flipped it on.*