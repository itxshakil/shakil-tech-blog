---
title: "Secure File Uploads: Seven Checks and Why Each One Exists"
datePublished: 2026-05-01T13:28:53.556Z
cuid: cmomy88q8005t1qjcd82m8w7f
slug: laravel-secure-file-upload
canonical: https://blog.shakiltech.com/laravel-secure-file-upload/
cover: https://cdn.hashnode.com/uploads/covers/63bac8d8624134a0c9b53b97/dc612acc-4574-4812-99a8-db38caaf33bf.png

---

A file upload is the moment you hand control to an untrusted user.

Everything else in your application — form inputs, query parameters, JSON — is text. You validate it, sanitize it, store it in a database. A file upload is arbitrary binary data from a source you cannot verify, about to be written to your filesystem. The surface area is different. The failure modes are different. The consequences are different.

Most tutorials cover the happy path: accept the file, store it, return a URL. The problem is never the happy path.

Rename a PHP file to `image.jpg`. Upload it. Many Laravel applications accept it — because `getClientOriginalExtension()` returns `jpg` and `getClientMimeType()` returns `image/jpeg`. Both values the *client* provided. Neither verified by the server.

This is how webshells get uploaded. This is how stored XSS gets planted in SVG files. This is the gap.

The fix is seven independent checks, each closing a specific attack vector. The reason for seven — not one — is that no single check catches everything. A determined attacker probes each layer individually. Defense in depth means removing the easy paths at each layer.

* * *

## What you will build

*   A middleware that validates seven independent properties of every uploaded file, each one named and explained
    
*   Server-side MIME detection using `finfo` — not the browser's claim
    
*   A unique name strategy that removes user-controlled strings from your filesystem paths entirely
    
*   A storage pattern where no uploaded file is ever directly reachable via HTTP
    

* * *

## The validation pipeline

```plaintext
Upload arrives
       │
       ▼
  1. Filename sanitization
     blocks: path traversal · double extensions · null bytes · hidden files
       │
       ▼
  2. Extension allowlist
     blocks: file types with no legitimate use case
       │
       ▼
  3. Client MIME allowlist
     blocks: mismatched browser-reported content type
       │
       ▼
  4. Server MIME detection (finfo reads actual bytes)
     blocks: renamed files · spoofed extensions
       │
       ▼
  5. Extension ↔ MIME cross-check
     blocks: shell.php.jpg · mismatched pairs
       │
       ▼
  6. File size limit
     blocks: denial-of-service via oversized uploads
       │
       ▼
  7. Magic bytes check
     blocks: crafted files · polyglot attacks
       │
       ▼
  Stored as unique name in storage/ — never in public/
```

* * *

## The middleware

The validation runs as middleware applied to upload routes. All validation aborts with `400` and a user-readable message:

```php
class SecureFileUpload
{
    // Only what your application genuinely needs — keep this narrow
    protected array $allowedExtensions = ['jpg', 'jpeg', 'png', 'pdf', 'docx', 'xlsx', 'csv'];

    protected array $allowedMimeTypes = [
        'image/jpeg', 'image/png', 'application/pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'text/csv', 'text/plain', 'application/vnd.ms-excel',
    ];

    // Maps each allowed extension to its permitted server-detected MIME types
    protected array $extensionMimeMap = [
        'jpg'  => ['image/jpeg'],
        'jpeg' => ['image/jpeg'],
        'png'  => ['image/png'],
        'pdf'  => ['application/pdf'],
        'docx' => ['application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
        'xlsx' => ['application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'],
        'csv'  => ['text/csv', 'text/plain', 'application/vnd.ms-excel'],
    ];

    protected int $maxFileSizeKb;

    public function __construct()
    {
        $this->maxFileSizeKb = (int) config('files.upload_max_kb', 5120); // Default 5MB
    }

    public function handle(Request $request, Closure $next)
    {
        foreach ($request->allFiles() as $file) {
            $this->walkFiles($file);
        }
        return $next($request);
    }

    protected function walkFiles($file): void
    {
        if (is_array($file)) {
            foreach ($file as $f) {
                $this->walkFiles($f);
            }
            return;
        }
        if ($file instanceof UploadedFile) {
            $this->validateFile($file);
        }
    }

    // Orchestrates all seven checks in order
    protected function validateFile(UploadedFile $file): void
    {
        $rawName    = $file->getClientOriginalName();
        $extension  = strtolower($file->getClientOriginalExtension());
        $clientMime = $file->getClientMimeType();

        // Check 1 — filename sanitization
        $this->sanitizeRawName($rawName);

        // Check 2 — extension allowlist
        if (!in_array($extension, $this->allowedExtensions, true)) {
            abort(400, "Files with .$extension extension are not supported.");
        }

        // Check 3 — client MIME allowlist
        if (!in_array($clientMime, $this->allowedMimeTypes, true)) {
            abort(400, 'Unrecognized file type. Ensure the file is not corrupted.');
        }

        // Check 4 — server-side MIME detection
        $serverMime = $this->detectServerMime($file);

        // Check 5 — extension ↔ MIME cross-check
        if (!$this->isMimeAllowedForExtension($extension, $serverMime)) {
            abort(400, "Extension '.$extension' does not match the file's content. "
                      . "Please ensure you have not renamed a file to a different extension.");
        }

        // Check 6 — size limit
        $sizeKb = (int) ceil($file->getSize() / 1024);
        if ($sizeKb > $this->maxFileSizeKb) {
            $maxMb  = round($this->maxFileSizeKb / 1024, 1);
            $fileMb = round($sizeKb / 1024, 1);
            abort(400, "File too large. Maximum is {$maxMb}MB. Your file is {$fileMb}MB.");
        }

        // Check 7 — magic bytes
        $this->lightweightMagicChecks($file, $extension);
    }

    protected function isMimeAllowedForExtension(string $extension, string $serverMime): bool
    {
        $allowed = $this->extensionMimeMap[$extension] ?? [];
        return in_array($serverMime, $allowed, true);
    }
}
```

The error messages matter. "Invalid file type" forces a support ticket. "Only one dot is allowed in the filename (e.g. `document.pdf`)" tells the user exactly what is wrong. When legitimate users trigger these checks by accident — and they will — specificity is the difference between a solvable problem and frustrated churn.

* * *

## Check 1 — Filename sanitization

Before any content check, validate the name itself. Filenames are a distinct attack surface:

```php
protected function sanitizeRawName(string $name): string
{
    if (mb_strlen($name, 'UTF-8') > 100) {
        abort(400, 'Filename too long. Please use 100 characters or fewer.');
    }

    // Path traversal: ../../etc/passwd
    if (preg_match('#[\\\\/]|^\.+$#', $name)) {
        abort(400, 'Filename cannot contain path separators.');
    }

    // Null bytes and control characters
    if (preg_match('/[\x00-\x1F\x7F]/', $name)) {
        abort(400, 'Filename contains invalid characters. Please rename the file.');
    }

    // Double-dot path traversal
    if (str_contains($name, '..')) {
        abort(400, "Filename cannot contain '..'.");
    }

    // Double-extension attack: shell.php.jpg
    // The browser sees .jpg, the server may execute .php
    if (substr_count($name, '.') > 1) {
        abort(400, "Only one dot allowed in the filename (e.g. 'document.pdf').");
    }

    // Hidden files: .htaccess, .env
    if (str_starts_with($name, '.')) {
        abort(400, 'Filenames cannot start with a dot.');
    }

    // Strict allowlist: letters, numbers, underscores, dashes, one dot, spaces
    if (!preg_match('/^[a-zA-Z0-9_\-\. ]+$/', $name)) {
        abort(400, 'Filename contains unsupported characters. Use letters, numbers, dashes, and underscores.');
    }

    return $name;
}
```

The double-extension rule is the highest-value check here. `shell.php.jpg` gets past a simple extension check because the last extension is `.jpg`. One dot, full stop.

* * *

## Checks 2 and 3 — Extension and client MIME allowlisting

Both are necessary, even though neither is sufficient alone:

```php
if (!in_array($extension, $this->allowedExtensions, true)) {
    abort(400, "Files with .$extension extension are not supported.");
}

if (!in_array($clientMime, $this->allowedMimeTypes, true)) {
    abort(400, 'Unrecognized file type. Ensure the file is not corrupted.');
}
```

**Why both?** An attacker who crafts a file with an allowed extension might supply an unexpected MIME type. An attacker who spoofs the MIME type might use a disallowed extension. Checking both raises the bar.

**Why keep the allowlist narrow?** Every extension you permit is an attack surface you are accepting. If your application does not need `.svg` uploads, remove it. If it does not need `.zip`, remove it. The default should be minimal; expansion should require a specific business case.

* * *

## Check 4 — Server-side MIME detection

This is the check that actually matters. The previous two trusted the client. This one reads the file:

```php
protected function detectServerMime(UploadedFile $file): string
{
    // finfo reads the file's actual bytes from the filesystem.
    // It does not care what the browser claimed.
    // A PHP file renamed to .jpg will be identified as text/x-php, not image/jpeg.
    $finfo = new \finfo(FILEINFO_MIME_TYPE);
    $mime  = $finfo->file($file->getRealPath());

    // Edge case: finfo correctly identifies CSV as text/plain (it is plain text).
    // But text/plain is not in the CSV slot on the allowlist.
    // The extension provides the context needed to resolve this.
    if ($mime === 'text/plain'
        && strtolower($file->getClientOriginalExtension()) === 'csv') {
        return 'text/csv';
    }

    return (string) $mime;
}
```

**Why does the CSV edge case matter?** A naive implementation would either reject all CSVs (wrong MIME type) or add `text/plain` to the global allowlist (which permits uploading arbitrary text files, including PHP in some environments). The explicit edge case handles the ambiguity without either problem.

* * *

## Check 5 — Cross-checking extension against detected MIME

The check that closes the most common attack vector. The `extensionMimeMap` defined on the class is what makes this work — each allowed extension maps to the MIME types `finfo` legitimately returns for that format:

```php
// CSV legitimately maps to multiple MIME types because different tools
// and operating systems report it differently. The map accommodates
// known variation without widening the gap for other formats.
'csv' => ['text/csv', 'text/plain', 'application/vnd.ms-excel'],
```

A file with extension `.jpg` whose `finfo`\-detected MIME is `text/x-php` fails this check. The extension says one thing; the bytes say another. That inconsistency is the signal.

* * *

## Check 6 — Size limits

The size check in `validateFile()` above reads the limit from config:

```php
$this->maxFileSizeKb = (int) config('files.upload_max_kb', 5120);
```

Store the limit in `config('files.upload_max_kb')` rather than hardcoding it. This lets you adjust per environment — tighter on free tier plans, looser for premium users — without touching the middleware. The error message surfaces both the maximum and the actual file size so the user knows exactly what to change.

* * *

## Check 7 — Magic bytes

The final layer reads the file's opening bytes directly:

```php
protected function lightweightMagicChecks(UploadedFile $file, string $ext): void
{
    $fp   = @fopen($file->getRealPath(), 'rb');
    $head = @fread($fp, 8) ?: '';
    @fclose($fp);

    // Every valid PDF begins with %PDF-
    if ($ext === 'pdf' && !str_starts_with($this->bytesToAscii($head), '%PDF-')) {
        abort(400, 'This PDF appears to be corrupted or invalid.');
    }

    // PNG has a fixed 8-byte signature defined in the spec
    if ($ext === 'png' && $head !== "\x89PNG\r\n\x1a\n") {
        abort(400, 'This PNG appears to be corrupted or invalid.');
    }

    // JPEG starts with FF D8 FF
    if (in_array($ext, ['jpg', 'jpeg'], true)) {
        if (!str_starts_with($head, "\xFF\xD8\xFF")) {
            abort(400, 'This JPEG appears to be corrupted or invalid.');
        }
    }

    // DOCX, XLSX, PPTX are ZIP archives with a PK header
    if (in_array($ext, ['docx', 'xlsx', 'pptx', 'zip'], true)) {
        if (!preg_match('/^PK(\x03\x04|\x05\x06|\x07\x08)/', $head)) {
            abort(400, 'This Office document appears corrupted or invalid.');
        }
    }
}

protected function bytesToAscii(string $bytes): string
{
    // Strip non-printable characters for string comparison (e.g. "%PDF-")
    return preg_replace('/[^\x20-\x7E]/', '', $bytes) ?? '';
}
```

**Why 8 bytes?** Magic byte signatures are typically 2–8 bytes. Reading the first 8 captures all common signatures without reading a meaningful chunk of the file.

**A note on coverage:** this check covers the formats most commonly exploited — PDF, PNG, JPEG, and Office documents. It is a structural sanity check, not a full file parser. Its job is to catch files where the opening bytes contradict the claimed format. It does not replace the MIME detection and cross-check above — all seven checks work together.

* * *

## SVG: the format that deserves extra attention

SVG files are XML. XML can contain `<script>` tags. A valid, well-formed SVG with embedded JavaScript is a stored XSS vector — if you accept it from one user and serve it inline to another, you have created a script injection path. The [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html) covers this and other format-specific risks in depth.

Three options:

1.  **Sanitize on upload** — strip script tags and event handlers before storing. Complex to do correctly and easy to get wrong.
    
2.  **Force download** — serve SVGs with `Content-Disposition: attachment`, preventing inline browser rendering and therefore script execution.
    
3.  **Remove SVGs from the allowlist** — the safest option if there is no specific business requirement.
    

If you do not have a defined reason to accept user-uploaded SVGs, option 3 takes one line and eliminates the risk entirely.

* * *

## Always generate a unique name before saving

Regardless of the validation outcome, the filename used for storage should never be the one the user provided:

```php
public static function uniqueName($file, $postfix = ''): string
{
    if ($file instanceof UploadedFile) {
        $file = $file->getClientOriginalName();
    }

    $extension = self::getExtension($file);

    $nameWithoutExtension = Str::of($file)
        ->replaceLast(".$extension", '')
        ->replace(' ', '_');

    if ($nameWithoutExtension->length() > 100) {
        $nameWithoutExtension = $nameWithoutExtension->substr(0, 100);
    }

    $nameWithoutExtension = $nameWithoutExtension . $postfix;

    if (!method_exists(Str::class, 'transliterate')) {
        $nameWithoutExtension = preg_replace('/[^\x20-\x7E]/', '_', $nameWithoutExtension);
        return $nameWithoutExtension . '_' . uniqid() . '.' . $extension;
    }

    return Str::transliterate($nameWithoutExtension) . '_' . uniqid() . '.' . $extension;
}
```

`uniqid()` is microsecond-based and not cryptographically random. It is sufficient here because it combines with the sanitized filename prefix — two simultaneous uploads of `report.pdf` produce `report_6601a1f3b8e42.pdf` and `report_6601a1f3b8e43.pdf`, with no collision. If your application requires stronger uniqueness guarantees — for example, if filenames are ever used as access tokens — substitute `Str::uuid()`. That is a one-line change and the stronger default.

`uniqueName()` depends on two helper methods. Here are simple implementations to drop into the same `File` class:

```php
public static function getExtension(string|UploadedFile $file): string
{
    if ($file instanceof UploadedFile) {
        return strtolower($file->getClientOriginalExtension());
    }

    return strtolower(pathinfo($file, PATHINFO_EXTENSION));
}

public static function getHash(UploadedFile $file): string
{
    // SHA-256 of the file contents — useful for detecting duplicate uploads
    // and verifying file integrity when serving back to users
    return hash_file('sha256', $file->getRealPath());
}
```

And in the media trait:

```php
public function addMediaAs(
    UploadedFile $file,
    string $collection = 'default',
    string $name = null
): Media {
    $uniqueName = $name ?? File::uniqueName($file);

    return $this->medias()->create([
        'name'       => $file->getClientOriginalName(), // Original name for display
        'file_name'  => $uniqueName,                    // Unique name for storage
        'mime_type'  => $file->getMimeType(),
        'path'       => $file->storeAs(
                            $this->getMediaPath($collection),
                            $uniqueName,
                            $this->getMediaDisk($collection)
                        ),
        'disk'       => $this->getMediaDisk($collection),
        'file_hash'  => File::getHash($file),
        'collection' => $collection,
        'size'       => $file->getSize(),
    ]);
}
```

**Why keep the original name in** `name` **but use the unique name for** `path`**?** The display name — shown to the user in the UI — should be what they uploaded. The storage path should have no connection to any user-provided string. The two concerns are separate.

* * *

## Where files live after validation

Store outside the web root — in `storage/` — and serve through a controller that enforces authorization:

```php
public function stream(Media $media): StreamedResponse
{
    // Authorization on every file access — no direct URL bypasses this
    $this->authorize('view', $media);

    return Storage::disk($media->disk)->response(
        $media->path,
        $media->name,
        ['Content-Type' => $media->mime_type]
    );
}
```

**Why never** `public/`**?** A file in `public/` is directly reachable via HTTP — no authorization, no application code in the path. If a file with a server-executable extension reaches `public/` because validation has layers that can be circumvented, it can be executed by guessing its URL. Files in `storage/` have no direct URL. Everything goes through the controller.

* * *

## Registering the middleware

Apply `SecureFileUpload` to upload routes only — never globally, since running all seven checks on every request that has no file would add overhead for no reason.

```php
// Laravel 11 — routes/web.php or routes/api.php
Route::post('/upload', [MediaController::class, 'store'])
    ->middleware(\App\Http\Middleware\SecureFileUpload::class);

// Or as a group if you have multiple upload endpoints
Route::middleware(\App\Http\Middleware\SecureFileUpload::class)->group(function () {
    Route::post('/media/upload', [MediaController::class, 'store']);
    Route::post('/documents/upload', [DocumentController::class, 'store']);
});

// Laravel 10 and earlier — same route syntax works, or alias it
// in app/Http/Kernel.php under $routeMiddleware:
// 'secure.upload' => \App\Http\Middleware\SecureFileUpload::class,
// Then use ->middleware('secure.upload') in your routes
```

* * *

> **The key insight from this article:** No single file validation check catches everything. `getClientMimeType()` trusts the browser. `getClientOriginalExtension()` trusts the browser. Only `finfo` reads the actual bytes. Seven independent checks layered together — each blocking a different attack path — is what makes upload validation meaningfully secure rather than just present.

## Before you ship — checklist

*   \[ \] `SecureFileUpload` middleware is applied to upload routes only — not globally
    
*   \[ \] `allowedExtensions` and `allowedMimeTypes` are minimal — only what the application genuinely needs
    
*   \[ \] `finfo` server-side MIME detection is running (not just trusting `getClientMimeType()`)
    
*   \[ \] `extensionMimeMap` covers every extension in `allowedExtensions`
    
*   \[ \] Magic bytes check covers JPEG in addition to PDF, PNG, and Office formats
    
*   \[ \] SVG is either removed from the allowlist or served with `Content-Disposition: attachment`
    
*   \[ \] `File::uniqueName()` is called before every `storeAs()` — original filename never used as storage path
    
*   \[ \] Files are stored in `storage/` — nothing uploaded goes into `public/`
    
*   \[ \] File access goes through a controller that calls `$this->authorize()`
    

* * *

*Previous:* [*Part 2 — Queue Architecture*](https://blog.shakiltech.com/laravel-queue-architecture/)