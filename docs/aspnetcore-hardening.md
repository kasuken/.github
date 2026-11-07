# ASP.NET Core hardening baseline

The minimum every .NET product (Brainy, LearnStack, MoneyBrain, Needly and any new app) should ship with before it is promoted. Each section is a copy-paste snippet plus the one thing it protects against. Snippets target .NET 10 and the minimal hosting model (`Program.cs`); adjust names to the app.

Reference implementation: Needly's [`SecurityHeadersExtensions.cs`](https://github.com/kasuken/Needly/blob/main/Needly.Web/Security/SecurityHeadersExtensions.cs) and [`RateLimitingExtensions.cs`](https://github.com/kasuken/Needly/blob/main/Needly.Web/Security/RateLimitingExtensions.cs).

## Checklist

- [ ] Forwarded headers configured, so scheme, client IP and rate limits are correct behind Azure
- [ ] Security headers on every response: CSP, `Referrer-Policy`, `X-Content-Type-Options`, `Permissions-Policy`, framing denied
- [ ] HSTS with a one year `max-age`
- [ ] Rate limiter on every anonymous or authentication endpoint
- [ ] Account lockout on failed sign-in (apps with local passwords)
- [ ] Every `BackgroundService` loop body catches and logs exceptions

## 1. Forwarded headers

On App Service the app sits behind a front end that terminates TLS. Without this, `Request.Scheme` is always `http` (breaking HTTPS redirects, secure cookies and OAuth redirect URIs) and `RemoteIpAddress` is the front end, so every client shares one rate-limit bucket.

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    // The App Service front end is not a fixed address, so the default loopback-only allow
    // list would ignore the headers. Only do this when the app is reachable solely through
    // the platform front end; otherwise list the real proxies in KnownProxies/KnownNetworks.
    options.KnownIPNetworks.Clear();
    options.KnownProxies.Clear();
});

// ...

var app = builder.Build();
app.UseForwardedHeaders(); // first in the pipeline, before HSTS, redirection, auth and rate limiting
```

Notes:

- On .NET 8 and 9 the property is named `KnownNetworks`; .NET 10 renames it to `KnownIPNetworks`.
- Do not clear the allow lists on a host that is directly reachable from the internet: any client could then spoof `X-Forwarded-For`.

## 2. Security headers

A small middleware keeps this in one reviewable place. The CSP below is strict (same-origin scripts, per-request nonce for the inline scripts Blazor and ASP.NET Core render). Extend `img-src`, `font-src`, `connect-src` and `style-src` only for origins the app actually uses.

```csharp
public static class SecurityHeadersExtensions
{
    private const string NonceItemKey = "App.CspNonce";

    private const string CspTemplate =
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{nonce}'; " +
        "style-src 'self'; " +           // add 'unsafe-inline' only if a component library needs it
        "img-src 'self' data:; " +
        "font-src 'self'; " +
        "connect-src 'self'; " +         // same-origin WebSockets (Blazor Server, SignalR) are covered
        "object-src 'none'; " +
        "base-uri 'self'; " +
        "form-action 'self'; " +
        "frame-ancestors 'none'";

    public static string? GetCspNonce(this HttpContext context) =>
        context.Items.TryGetValue(NonceItemKey, out var nonce) ? nonce as string : null;

    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app) =>
        app.Use(async (context, next) =>
        {
            context.Items[NonceItemKey] = Convert.ToBase64String(RandomNumberGenerator.GetBytes(16));
            context.Response.OnStarting(static state =>
            {
                var http = (HttpContext)state;
                var headers = http.Response.Headers;
                // Assign rather than Add: some framework components set their own, weaker values.
                headers.ContentSecurityPolicy = CspTemplate.Replace("{nonce}", http.GetCspNonce() ?? "");
                headers.XContentTypeOptions = "nosniff";
                headers.XFrameOptions = "DENY";
                headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
                headers["Permissions-Policy"] =
                    "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()";
                headers["Cross-Origin-Opener-Policy"] = "same-origin";
                return Task.CompletedTask;
            }, context);
            await next(context);
        });
}
```

Use the nonce on any inline script the app emits: `<script nonce="@Context.GetCspNonce()">`. Roll a new CSP out with `Content-Security-Policy-Report-Only` first if the app has many third-party origins.

Static sites on Azure Static Web Apps set the same headers under `globalHeaders` in `staticwebapp.config.json` instead.

## 3. HSTS

Outside Development only. One year is the baseline; add `preload` only when every subdomain is HTTPS-only for good.

```csharp
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true; // drop if any subdomain is still served over HTTP
});

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
app.UseHttpsRedirection();
```

The default `max-age` is 30 days, which is too short to protect returning visitors.

## 4. Rate limiting on authentication and anonymous endpoints

Partition by client IP (correct only once section 1 is in place). Keep limits generous enough for real users and strict enough to stop credential stuffing; tune per endpoint.

```csharp
builder.Services.AddRateLimiter(limiter =>
{
    limiter.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    limiter.OnRejected = (context, _) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)Math.Ceiling(retryAfter.TotalSeconds)).ToString(CultureInfo.InvariantCulture);
        }
        return ValueTask.CompletedTask;
    };

    limiter.AddPolicy("auth", context => RateLimitPartition.GetFixedWindowLimiter(
        (context.Connection.RemoteIpAddress ?? IPAddress.None).ToString(),
        _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 20,
            Window = TimeSpan.FromMinutes(1),
            QueueLimit = 0,
        }));
});

// ...

app.UseForwardedHeaders();
app.UseRateLimiter();            // after UseRouting when endpoint routing is used explicitly

app.MapGet("/auth/login", LoginHandler).RequireRateLimiting("auth");
app.MapPost("/auth/register", RegisterHandler).RequireRateLimiting("auth");
// Controllers / Razor Pages: [EnableRateLimiting("auth")]
```

Cover at least: sign-in, registration, password reset, email confirmation resend, webhook receivers and any public form. Webhooks should also verify a signature; the limit is a second layer.

## 5. Lockout on failed sign-in

For apps that use ASP.NET Core Identity with local passwords. `lockoutOnFailure` defaults to `false`, so lockout silently does nothing unless the sign-in call opts in.

```csharp
builder.Services.AddIdentity<AppUser, IdentityRole>(options =>
{
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;
});

var result = await signInManager.PasswordSignInAsync(
    userName, password, isPersistent, lockoutOnFailure: true);   // not false

if (result.IsLockedOut)
{
    // Return the same generic message as a wrong password, to avoid account enumeration.
}
```

The same applies to `CheckPasswordSignInAsync(user, password, lockoutOnFailure: true)` and to two-factor and recovery-code sign-in. Apps that sign in only through an external provider (GitHub, Microsoft) have no local password to lock out.

## 6. BackgroundService exception handling

Since .NET 6 the default `BackgroundServiceExceptionBehavior` is `StopHost`: one unhandled exception in `ExecuteAsync` (a SQL timeout, an HTTP blip) ends the whole web app, and with a durable work queue the same item can be replayed on every restart into a crash loop. Catch inside the loop, log, and let the next iteration retry. Let cancellation through so shutdown stays clean.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var timer = new PeriodicTimer(interval, timeProvider);
    do
    {
        try
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            await scope.ServiceProvider.GetRequiredService<IWorker>().RunAsync(stoppingToken);
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            throw; // normal shutdown
        }
        catch (Exception exception)
        {
            logger.LogError(exception, "Worker pass failed; retrying on the next interval");
        }
    }
    while (await timer.WaitForNextTickAsync(stoppingToken));
}
```

For queue consumers, wrap each item the same way so one poison message is logged and skipped rather than stopping the loop. When the handler uses an EF Core `DbContext`, a failed `SaveChangesAsync` leaves the failing entities tracked: before recording the failure on the same context, detach them (`ChangeTracker.Clear()` or set their state to `Detached`), otherwise the next `SaveChangesAsync` replays the same failing statement and throws again.

Do not switch the host to `BackgroundServiceExceptionBehavior.Ignore` as a shortcut: it hides a worker that has died for good. Use it only together with the loop-level handling above.

## Verifying

- `curl -sI https://<app>/` shows the headers from sections 2 and 3.
- Hit a rate-limited endpoint above its limit and expect `429` with `Retry-After`; behind Azure, check that two different clients do not share one bucket (section 1).
- Add a test that a worker pass that throws is logged and does not complete the `ExecuteAsync` task.
- [securityheaders.com](https://securityheaders.com) and the browser console (CSP violations) are quick external checks.

## Rollout status

Applying this baseline to each product is tracked in that product's repository, not here.
