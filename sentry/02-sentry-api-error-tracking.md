# Sentry for API Request & Error Tracking — Dalgo Engineering Guide

A practical, copy-paste guide to set up and run Sentry across the Dalgo stack
(Django + Django Ninja backend, Next.js frontend, FastAPI prefect-proxy), tuned
for a small team, NGO data sensitivity, and a tight budget.

> **What's already in place (grounded in the repos as of this writing):**
> - **`DDP_backend`** — `sentry-sdk==2.39.0` is installed and **already initialised**
>   in `ddpui/settings.py` with `DjangoIntegration` + `LoggingIntegration`. There is a
>   **real gap** in `ddpui/routes.py` (catch-all Ninja handler) covered in §2.3 — fix this first.
> - **`webapp_v2`** — `@sentry/nextjs ^10.53.1` is **already a dependency**. Verify the
>   instrumentation files exist (§3) and that the DSN/auth token are wired.
> - **`prefect-proxy`** — no Sentry yet. Greenfield (§2.6).

---

## 1. Sentry fundamentals for this stack

### The four data products
| Product | What it captures | Why Dalgo cares |
|---------|------------------|-----------------|
| **Errors** | Unhandled + manually captured exceptions, with stack trace, request, breadcrumbs | "An Airbyte sync API call 500'd for org `akvo`" — the core ask |
| **Performance / Tracing** | Timed `transactions` (one per request) made of `spans` (DB query, HTTP call) | "Why is `/api/dashboard` slow for this NGO?" Slow DB + Prefect/Airbyte calls |
| **Logs** | Structured log lines routed to Sentry (`enable_logs`) | Correlate a log line with the error/trace that produced it |
| **Session Replay** | Privacy-masked DOM recording of the browser session | Reproduce a UI bug a field-staff user hit on an old device — **mask aggressively** (§3.4) |

### The mental model
- **Issue** — a *group* of similar events (e.g. all `ZeroDivisionError` at the same code line). You triage issues, not events.
- **Event** — one occurrence of an error or one transaction.
- **Transaction** — a timed unit of work, usually one HTTP request. Named like `GET /api/airbyte/connections`.
- **Span** — a timed sub-operation inside a transaction (a SQL query, an outbound call to the Prefect API).
- **Trace** — a tree of transactions/spans spanning *services*. One browser click → Next.js → Django → prefect-proxy is **one trace** (§4).
- **Release** — a version of your code (a git SHA). Errors get tagged with the release so Sentry can show *suspect commits* and *release health* (§5.3).
- **Environment** — `dev` / `staging` / `production`. Filter and alert per environment.

Docs: [Tracing](https://docs.sentry.io/product/sentry-basics/tracing/) ·
[Releases](https://docs.sentry.io/product/releases/) ·
[Environments](https://docs.sentry.io/product/sentry-basics/environments/)

---

## 2. Backend: Django + Django Ninja (`DDP_backend`)

### 2.1 Current init (already in `ddpui/settings.py`)
The existing block is good. Here it is annotated with the few additions you want
(`release`, `traces_sampler`, `before_send`). Replace the current `sentry_sdk.init(...)`:

```python
# ddpui/settings.py
import os
import logging
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.logging import LoggingIntegration

from ddpui.utils.sentry_filters import before_send, traces_sampler  # new, see §2.5 / §6.1

sentry_sdk.init(
    dsn=os.getenv("SENTRY_DSN"),
    integrations=[
        DjangoIntegration(
            transaction_style="url",   # transactions named "GET /api/airbyte/connections/{id}"
            middleware_spans=True,     # see time spent in middleware (auth, CORS)
            cache_spans=True,          # Redis cache timing
            signals_spans=False,       # noise reduction; flip on only when debugging signals
        ),
        # INFO+ logs become breadcrumbs; ERROR+ logs become Sentry issues.
        LoggingIntegration(level=logging.INFO, event_level=logging.ERROR),
    ],
    # Performance. Prefer a sampler over a flat rate so you can sample health-checks at 0. §6.1
    traces_sampler=traces_sampler,
    profiles_sample_rate=float(os.getenv("SENTRY_PSR", "0.1")),
    enable_logs=os.getenv("SENTRY_ENABLE_LOGS", "True") == "True",
    # PII: see §2.5. Keep True ONLY with the before_send scrubber wired below.
    send_default_pii=os.getenv("SENTRY_SEND_DEFAULT_PII", "False") == "True",
    environment=os.getenv("ENVIRONMENT", "staging"),
    # Tie every event to a deploy → suspect commits + release health. §5.3
    release=os.getenv("SENTRY_RELEASE"),  # CI sets this to the git SHA
    before_send=before_send,              # PII scrubbing + noise filtering
)
```

Reference: [Sentry Django guide](https://docs.sentry.io/platforms/python/guides/django/) ·
[Options](https://docs.sentry.io/platforms/python/configuration/options/)

### 2.2 How unhandled exceptions are captured
For a *plain Django view*, the `DjangoIntegration` hooks Django's `got_request_exception`
signal — any exception that bubbles out of a view is auto-captured before Django renders
its 500 page. You don't write any code for that path.

`LoggingIntegration(event_level=logging.ERROR)` is a second capture path: any
`logger.error(...)` / `logger.exception(...)` becomes a Sentry issue. Dalgo's
`unified_logger` formats `exc_info`, so logged exceptions arrive with a stack trace.

### 2.3 Django Ninja specifics — the important gap

**Django Ninja consumes exceptions.** When a registered
`@api.exception_handler(...)` returns a `Response`, Ninja has *handled* the request —
the exception never bubbles out to Django, so `got_request_exception` never fires, so
the **`DjangoIntegration` auto-capture never sees it**
([Django Ninja errors](https://django-ninja.dev/guides/errors/)).

Dalgo's `ddpui/routes.py` has exactly this catch-all:

```python
@src_api.exception_handler(Exception)
def ninja_default_error_handler(request, exc: Exception):
    """Handle any other exception raised in the apis"""
    return Response({"detail": str(exc)}, status=500)
```

This returns a 500 to the client but **does not report to Sentry**. Every unexpected
500 in the API is currently invisible in Sentry unless some view happened to
`logger.error(..., exc_info=True)` on the way down. **This is the #1 fix.** Make the
catch-all capture explicitly:

```python
# ddpui/routes.py
import sentry_sdk
from ninja.responses import Response

@src_api.exception_handler(Exception)
def ninja_default_error_handler(request, exc: Exception):
    """Report to Sentry, THEN return a clean 500 to the client."""
    sentry_sdk.capture_exception(exc)   # <-- the missing line
    return Response({"detail": "Something went wrong, our team has been notified."}, status=500)
```

Notes:
- Don't leak `str(exc)` to non-technical NGO users — it can carry internals/PII. Return a friendly message; the detail lives in Sentry.
- Keep the existing `ValidationError` (422) and `PydanticValidationError` (500) handlers as-is, but consider **not** reporting 422s (expected client errors — noise). Only `capture_exception` in the catch-all and in the Pydantic *response*-validation handler (that one is a real server bug).

### 2.4 Capturing handled errors with context
When you catch an exception but want it in Sentry (e.g. a degraded-but-recoverable
Airbyte call), capture it inside a scope so it carries org/feature context:

```python
import sentry_sdk

try:
    resp = airbyte_service.get_connections(workspace_id)
except AirbyteError as exc:
    with sentry_sdk.new_scope() as scope:
        scope.set_tag("feature_area", "airbyte")
        scope.set_tag("org_slug", orguser.org.slug)
        scope.set_context("airbyte", {"workspace_id": workspace_id})
        scope.set_level("warning")
        sentry_sdk.capture_exception(exc)
    raise  # or degrade gracefully
```

### 2.5 PII scrubbing — required before turning on `send_default_pii`
NGO request bodies can contain beneficiary data. Two layers:

1. **Identify users by internal id only** — never email — via middleware/auth:

```python
import sentry_sdk

# in CustomJwtAuthMiddleware after resolving the user/org
sentry_sdk.set_user({"id": str(orguser.id)})          # internal id, not email
sentry_sdk.set_tag("org_slug", orguser.org.slug)       # NGO tenant — filter by this everywhere
sentry_sdk.set_tag("user_role", orguser.new_role.slug if orguser.new_role else "unknown")
```

2. **A `before_send` scrubber** that strips request bodies, known sensitive keys, and
   query strings. Create `ddpui/utils/sentry_filters.py`:

```python
# ddpui/utils/sentry_filters.py
SENSITIVE_KEYS = {
    "password", "token", "secret", "authorization", "api_key", "apikey",
    "access_token", "refresh_token", "connection_configuration", "credentials",
    "email", "phone", "aadhaar", "beneficiary", "name",  # NGO data fields
}

def _scrub(obj):
    if isinstance(obj, dict):
        return {
            k: ("[Filtered]" if k.lower() in SENSITIVE_KEYS else _scrub(v))
            for k, v in obj.items()
        }
    if isinstance(obj, list):
        return [_scrub(v) for v in obj]
    return obj

def before_send(event, hint):
    # 1. Drop the raw request body entirely — NGO payloads can hold beneficiary data.
    request = event.get("request")
    if request:
        request.pop("data", None)                      # POST/PUT body
        if "cookies" in request:
            request["cookies"] = "[Filtered]"
        # strip query string from the URL (can carry tokens / ids)
        if "url" in request:
            request["url"] = request["url"].split("?")[0]
        # scrub known headers (DjangoIntegration scrubs Authorization by default,
        # but be explicit for custom headers)
        headers = request.get("headers", {})
        for h in list(headers):
            if h.lower() in SENSITIVE_KEYS or h.lower() == "x-api-key":
                headers[h] = "[Filtered]"

    # 2. Recursively scrub any custom contexts we attached.
    for ctx_name, ctx in (event.get("contexts") or {}).items():
        event["contexts"][ctx_name] = _scrub(ctx)

    # 3. Noise reduction — drop known-benign exceptions (§6.2).
    exc_type = (hint or {}).get("exc_info", (None,))[0]
    if exc_type and exc_type.__name__ in {"Http404", "PermissionDenied"}:
        return None  # don't send

    return event
```

The SDK also ships an automatic `EventScrubber` (denylists passwords/cookies/CSRF) and
`DjangoIntegration` scrubs `Authorization` headers — `before_send` is the
*Dalgo-specific* layer on top.
Refs: [Sensitive data](https://docs.sentry.io/platforms/python/data-management/sensitive-data/) ·
[Data collected](https://docs.sentry.io/platforms/python/data-management/data-collected/)

> **Recommendation:** keep `send_default_pii=False` in production. You still get the
> internal `user.id` you set manually, without IP addresses, cookies, or raw bodies.

### 2.6 The prefect-proxy (FastAPI)
Greenfield. Add `sentry-sdk` and init at app startup (before `FastAPI()` if possible):

```python
# prefect-proxy main / app factory
import os
import sentry_sdk
from sentry_sdk.integrations.starlette import StarletteIntegration
from sentry_sdk.integrations.fastapi import FastApiIntegration

sentry_sdk.init(
    dsn=os.getenv("SENTRY_DSN"),
    integrations=[
        StarletteIntegration(transaction_style="endpoint"),
        FastApiIntegration(
            transaction_style="endpoint",
            # report 5xx AND 403 as issues; 4xx client errors stay quiet
            failed_request_status_codes={403, *range(500, 600)},
        ),
    ],
    traces_sample_rate=float(os.getenv("SENTRY_TSR", "0.1")),
    send_default_pii=False,
    environment=os.getenv("ENVIRONMENT", "staging"),
    release=os.getenv("SENTRY_RELEASE"),
)
```

FastAPI auto-captures unhandled route exceptions; no decorator needed.
Ref: [Sentry FastAPI guide](https://docs.sentry.io/platforms/python/guides/fastapi/).

---

## 3. Frontend: Next.js (`webapp_v2`)

`@sentry/nextjs ^10.53.1` is installed. Run the wizard once to generate the
instrumentation files if they aren't present, then edit them:

```bash
npx @sentry/wizard@latest -i nextjs
```

### 3.1 The config files (App Router, SDK v10)
**`instrumentation-client.ts`** (browser):
```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_ENVIRONMENT ?? "staging",
  release: process.env.NEXT_PUBLIC_SENTRY_RELEASE,

  tracesSampleRate: process.env.NODE_ENV === "development" ? 1.0 : 0.1,

  // Link browser traces → Django/proxy. MUST include your API origins. §4.2
  tracePropagationTargets: [
    "localhost",
    /^https:\/\/api(\.staging)?\.dalgo\.org/,
  ],

  // Replay — heavily masked for NGO privacy. §3.4
  replaysSessionSampleRate: 0.0,   // don't record normal sessions (cost + privacy)
  replaysOnErrorSampleRate: 0.2,   // record a fraction of error sessions only
  integrations: [
    Sentry.replayIntegration({ maskAllText: true, blockAllMedia: true }),
    Sentry.browserTracingIntegration(),
  ],
});
```

**`sentry.server.config.ts`** and **`sentry.edge.config.ts`** — same `dsn`, `environment`,
`release`, `tracesSampleRate`; no replay.

**`instrumentation.ts`** (loads server/edge configs + request-error hook):
```typescript
import * as Sentry from "@sentry/nextjs";

export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") await import("./sentry.server.config");
  if (process.env.NEXT_RUNTIME === "edge") await import("./sentry.edge.config");
}
export const onRequestError = Sentry.captureRequestError;
```

**`next.config.ts`** — wrap with `withSentryConfig` (uploads source maps at build):
```typescript
import { withSentryConfig } from "@sentry/nextjs";
export default withSentryConfig(nextConfig, {
  org: "dalgo",
  project: "webapp-v2",
  authToken: process.env.SENTRY_AUTH_TOKEN,  // CI only, never commit
  silent: !process.env.CI,
});
```

**`app/global-error.tsx`** — React render-crash boundary that reports to Sentry
(see the wizard output / [Next.js guide](https://docs.sentry.io/platforms/javascript/guides/nextjs/)).

### 3.2 Capturing API request failures from the browser
Dalgo uses `swr`. Centralise capture in the fetcher so every failed API call is reported
once, with the endpoint as context:

```typescript
import * as Sentry from "@sentry/nextjs";

export async function apiFetcher(url: string, init?: RequestInit) {
  const res = await fetch(url, init);
  if (!res.ok) {
    Sentry.captureMessage(`API ${res.status} ${init?.method ?? "GET"} ${url}`, {
      level: res.status >= 500 ? "error" : "warning",
      tags: { http_status: String(res.status), endpoint: new URL(url).pathname },
    });
    throw new Error(`Request failed: ${res.status}`);
  }
  return res.json();
}
```
Because the browser SDK already instruments `fetch`, the failing request also appears as
a span/breadcrumb automatically — this just turns it into an actionable issue.

### 3.3 Source maps
Handled by `withSentryConfig` + `SENTRY_AUTH_TOKEN` at build time. Without it, prod stack
traces are minified garbage. Verify the `.env.sentry-build-plugin` (wizard-generated) is
git-ignored and that CI exports `SENTRY_AUTH_TOKEN`.

### 3.4 Session Replay — privacy for NGO users
- `maskAllText: true` + `blockAllMedia: true` — masks all text/inputs/media by default.
- Record **error sessions only** (`replaysSessionSampleRate: 0`), and only a fraction.
- This protects beneficiary data shown on screen while still letting you reproduce bugs.

### 3.5 Core Web Vitals on slow devices
`browserTracingIntegration()` auto-collects LCP / CLS / INP / FCP / TTFB. This is directly
relevant to your users on old devices and slow links — build a **Web Vitals dashboard**
filtered to `production` and watch p75 LCP/INP. Set a latency alert if p75 LCP regresses.

---

## 4. Tracking API requests well (the core ask)

### 4.1 What a full trace looks like
```
Trace (one trace_id)
└─ Browser txn: navigation /pipelines              [@sentry/nextjs]
   └─ HTTP span: GET https://api.dalgo.org/api/airbyte/connections
      └─ Django txn: GET /api/airbyte/connections/  [DjangoIntegration]
         ├─ db span: SELECT ... FROM orgs            (slow query shows here)
         └─ HTTP span: GET prefect-proxy/deployments
            └─ FastAPI txn: GET /proxy/deployments   [FastApiIntegration]
               └─ HTTP span: Prefect API call
```
You get this *for free* once tracing is on in all three services **and** the
`sentry-trace` + `baggage` headers propagate (§4.2).

### 4.2 Linking the traces — `tracePropagationTargets` / `trace_propagation_targets`
The browser SDK adds `sentry-trace` and `baggage` headers to outbound requests whose URL
matches `tracePropagationTargets`. Django/FastAPI read those headers and **continue the
same trace**. Two requirements:

1. **Frontend** lists your API origins (done in §3.1).
2. **CORS** must allow the headers. `ddpui/settings.py` already imports `default_headers`
   and almost certainly defines a `CORS_ALLOW_HEADERS` — **append** to it rather than
   overwriting, or browsers will strip the trace headers:

```python
# ddpui/settings.py — add to the EXISTING CORS_ALLOW_HEADERS tuple, don't replace it
CORS_ALLOW_HEADERS = (*CORS_ALLOW_HEADERS, "sentry-trace", "baggage")
# (or, if you build it from default_headers: (*default_headers, ..., "sentry-trace", "baggage"))
```

Backend Python SDKs default `trace_propagation_targets=['.*']` — they propagate to all
outbound calls (including Prefect/Airbyte), so the Django→proxy hop links automatically.
Refs: [trace propagation (Next.js)](https://docs.sentry.io/platforms/javascript/guides/nextjs/tracing/trace-propagation/) ·
[options](https://docs.sentry.io/platforms/python/configuration/options/).

### 4.3 A good API span + instrumenting external calls
`DjangoIntegration(cache_spans=True)` + DB spans already cover SQL and cache. Wrap the
expensive external calls (Airbyte / Prefect) so they show as named spans:

```python
import sentry_sdk

with sentry_sdk.start_span(op="http.client", name="airbyte.list_connections") as span:
    span.set_data("workspace_id", workspace_id)
    connections = airbyte_service.get_connections(workspace_id)
    span.set_data("count", len(connections))
```

Now a slow `/api/airbyte/...` transaction visibly attributes time to "airbyte" vs DB vs
your own code.

### 4.4 Capturing request/response context safely
- Let the SDK attach request metadata, but rely on `before_send` (§2.5) to strip the body.
- For response context, attach only shape/size, never the payload:
  `scope.set_context("response", {"status": status, "rows": len(rows)})`.

### 4.5 Performance alerts
- **p95 latency per endpoint** — metric alert on transaction duration, grouped by transaction.
- **Error rate per endpoint** — `failure_rate()` alert > threshold for a transaction.
- **Throughput drop** — useful for catching a dead Airbyte/Prefect dependency.
See §5.6 for the concrete alert table.

---

## 5. What to CREATE in Sentry

### 5.1 Projects
One project per deployable service — keeps issues, releases, and quotas separable:

| Project slug | Service | SDK |
|--------------|---------|-----|
| `ddp-backend` | Django + Ninja | `sentry-sdk` |
| `prefect-proxy` | FastAPI | `sentry-sdk` |
| `webapp-v2` | Next.js | `@sentry/nextjs` |

(They still link into shared **traces** across projects — separate projects ≠ separate traces.)

### 5.2 Environments
`development`, `staging`, `production`. Drive from the existing `ENVIRONMENT` env var
(already read in settings.py). Alert almost exclusively on `production`.

### 5.3 Releases + commit tracking in CI
Set `release` to the git SHA in every service (env vars above). Then, in each service's
deploy CI job, create+associate+finalize the release so Sentry can show **suspect commits**
and **release health**. Using `sentry-cli`:

```bash
export SENTRY_AUTH_TOKEN=...        # CI secret
export SENTRY_ORG=dalgo
VERSION=$(sentry-cli releases propose-version)   # = git SHA

sentry-cli releases new -p ddp-backend "$VERSION"
sentry-cli releases set-commits --auto "$VERSION"   # needs the GitHub integration installed
sentry-cli releases finalize "$VERSION"
sentry-cli releases deploys "$VERSION" new -e production
```

Or the GitHub Action (frontend builds, uploads source maps, and registers the release):
```yaml
- uses: getsentry/action-release@v3
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
    SENTRY_ORG: dalgo
    SENTRY_PROJECT: webapp-v2
  with:
    environment: production
    version: ${{ github.sha }}
```
Install the **Sentry ↔ GitHub integration** once so `set-commits --auto` can map a SHA to
its commits.
Refs: [Release setup](https://docs.sentry.io/product/releases/setup/) ·
[Associate commits](https://docs.sentry.io/product/releases/associate-commits/).

### 5.4 Custom tags worth adding
| Tag | Source | Use |
|-----|--------|-----|
| `org_slug` | auth middleware | **Most important.** "Is this bug one NGO or all of them?" |
| `user_role` | orguser role | account-admin vs pipeline-manager vs report-viewer |
| `feature_area` | set per router/handler | airbyte / dbt / dashboard / pipeline / charts |
| `http_status` | frontend fetcher | filter 4xx vs 5xx |
| `release` | CI | regression triage |

### 5.5 Dashboards worth building
| Dashboard | Widgets |
|-----------|---------|
| **API health** | error rate by endpoint, p95 latency by endpoint, throughput |
| **Slowest endpoints** | top transactions by p95 duration (find the Airbyte/Prefect-bound ones) |
| **Errors by org** | issue count grouped by `org_slug` (is one NGO hurting?) |
| **Release health** | crash-free sessions/users per release, new issues per release |
| **Web Vitals** | p75 LCP / INP / CLS, `production` only — the slow-device lens |

### 5.6 Alerts worth setting
| Alert | Condition | Route to |
|-------|-----------|----------|
| New issue in prod | first seen, `environment:production` | Slack channel |
| Error spike | issue freq > N in 1h, or % spike vs baseline | Slack + on-call |
| Latency regression | transaction p95 > threshold (per critical endpoint) | Slack |
| Endpoint error rate | `failure_rate()` > 5% for an endpoint over 10m | Slack |
| Crash-free drop | crash-free sessions < 99% on a release | Slack + block rollout |

### 5.7 Custom fingerprinting for noisy errors
Collapse errors that share a root cause but differ in message (e.g. per-org Airbyte
timeouts) into one issue:
```python
def before_send(event, hint):
    # ... scrubbing from §2.5 ...
    exc = (hint or {}).get("exc_info", (None,))[0]
    if exc and exc.__name__ == "AirbyteConnectionError":
        event["fingerprint"] = ["airbyte-connection-error"]
    return event
```

---

## 6. Best practices

### 6.1 Sampling strategy (budget-aware)
You're on a tight quota. **Errors: keep 100%** (they're cheap and the whole point).
**Traces: sample low and smart** with a `traces_sampler` instead of a flat rate:

```python
# ddpui/utils/sentry_filters.py
def traces_sampler(ctx):
    name = (ctx.get("transaction_context") or {}).get("name", "")
    if any(p in name for p in ("/health", "/metrics", "favicon")):
        return 0.0                     # never trace noise
    if name.startswith("GET /api/airbyte") or "pipeline" in name:
        return 0.3                     # trace the endpoints you care about more
    return float(os.getenv("SENTRY_TSR", "0.05"))
```
Profiling: keep `profiles_sample_rate` low (0.1) in prod. Replay: error-only.

### 6.2 Noise reduction
- Drop expected exceptions in `before_send` (`Http404`, `PermissionDenied`, validation 422s).
- Use **Inbound Filters** (project settings UI) for browser noise: legacy browsers, known
  third-party/extension errors, localhost.
- Don't capture 422/4xx client errors as issues — they're user input, not bugs.

### 6.3 Alert fatigue avoidance
- Alert on **production only**, on **actionable** conditions (new issue, spike, regression) — not every event.
- Route to **one** Slack channel; escalate only spikes/crash-free drops.
- Use issue-frequency thresholds, not "any new event."
- Set **issue owners** (CODEOWNERS-style ownership rules) so alerts route to the right person.

### 6.4 Triage workflow for a small team
- **Daily (5 min):** scan new + escalating prod issues. Assign or resolve-as-ignored.
- **On a real issue:** assign owner → check suspect commit (from release) → link to a ticket.
- **Weekly (30 min):** review top issues by `org_slug` and slowest endpoints; clear stale ignores.
- **SLA guideline:** new prod error spiking → look within the hour; single new error → same day.
- **"Good" looks like:** every prod 500 produces a Sentry issue with org + release context,
  routed to a person, with a low steady alert volume nobody has learned to ignore.
Refs: [Sentry alerting best practices](https://blog.sentry.io/automate-group-and-get-alerted-a-best-practices-guide-to-monitoring-your/).

---

## 7. Rollout plan (prioritised)

**Phase 0 — close the blind spot (do first, ~1 hr)**
1. Add `sentry_sdk.capture_exception(exc)` to the catch-all Ninja handler in `routes.py` (§2.3) and stop leaking `str(exc)` to users.
2. Confirm `SENTRY_DSN` + `ENVIRONMENT` are set in staging/prod backend.

**Phase 1 — make it safe (~half day)**
3. Add `ddpui/utils/sentry_filters.py` with `before_send` + `traces_sampler`; wire into init.
4. Set `send_default_pii=False` in prod; set `user.id` + `org_slug` + `user_role` tags in auth middleware.
5. Add `sentry-trace`/`baggage` to `CORS_ALLOW_HEADERS`.

**Phase 2 — frontend + proxy (~1 day)**
6. Verify/generate Next.js instrumentation files; set `tracePropagationTargets` to API origins; wire `SENTRY_AUTH_TOKEN` in CI for source maps; mask replay.
7. Add Sentry to prefect-proxy (§2.6).

**Phase 3 — releases + visibility (~1 day)**
8. Install GitHub integration; add release create/set-commits/finalize to all three CI deploy jobs; set `release` env vars.
9. Build the API-health, errors-by-org, and release-health dashboards.

**Phase 4 — tune (ongoing)**
10. Add prod alerts (new-issue, spike, latency, crash-free) routed to one Slack channel.
11. After a week, prune noisy issues with fingerprints + inbound filters; lock in the sampler rates against actual quota usage.

---

### Sources
- [Sentry Django guide](https://docs.sentry.io/platforms/python/guides/django/) · [Django HTTP errors](https://docs.sentry.io/platforms/python/integrations/django/http_errors/)
- [Sentry FastAPI guide](https://docs.sentry.io/platforms/python/guides/fastapi/)
- [Sentry Next.js guide](https://docs.sentry.io/platforms/javascript/guides/nextjs/) · [Next.js trace propagation](https://docs.sentry.io/platforms/javascript/guides/nextjs/tracing/trace-propagation/)
- [Python options](https://docs.sentry.io/platforms/python/configuration/options/) · [Sensitive data](https://docs.sentry.io/platforms/python/data-management/sensitive-data/) · [Data collected](https://docs.sentry.io/platforms/python/data-management/data-collected/)
- [Releases setup](https://docs.sentry.io/product/releases/setup/) · [Associate commits](https://docs.sentry.io/product/releases/associate-commits/)
- [Django Ninja errors](https://django-ninja.dev/guides/errors/)
- [Sentry alerting best practices (blog)](https://blog.sentry.io/automate-group-and-get-alerted-a-best-practices-guide-to-monitoring-your/)
