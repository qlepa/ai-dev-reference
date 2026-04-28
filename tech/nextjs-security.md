# Next.js Security — Audit Reference
> Attach alongside `nextjs.md` when auditing or hardening a Next.js project.
> Part 1 covers the technical security posture. Part 2 covers AI-readiness — what instructions
> and agents to leave so AI never reintroduces a patched vulnerability.
>
> **How to use:** Open the project, attach this file, and ask:
> *"Audit this project against every section. For each item report: current state, risk level (Critical/High/Medium/Low), and recommended fix."*
> At the end ask Claude to produce `security-audit.md` with traffic-light status and a prioritized remediation list.
>
> If no base file (`fresh.md`, `new.md`, or `legacy.md`) is attached alongside this one, run a standalone audit: treat each section's guidelines as a checklist, assess the current state (✅ pass / ⚠️ partial / ❌ missing), and save a report as `nextjs-security-audit-YYYY-MM-DD.md`.
>
> **Audit strategy — start here:**
> 1. `next.config.ts` — headers(), redirects, CSP
> 2. `middleware.ts` — auth guards, rate limiting
> 3. `src/app/api/**/route.ts` — every Route Handler
> 4. Any file that reads `cookies()`, `headers()`, or `session`
> 5. Auth context / provider files
> 6. Files that write to `localStorage`, `sessionStorage`, `document.cookie`
> 7. `package.json` — run `pnpm audit` and scan for known-vulnerable dependencies

---

## Part 1 — Technical Security Audit

---

## 1. Authentication & Session Management

This is the highest-risk area in most Next.js apps. Each gap is a potential account takeover.

### 1.1 Token Storage

- [ ] **Auth tokens are stored in `HttpOnly` cookies, not `localStorage`**
  - `localStorage` is readable by any JavaScript on the page — XSS, supply chain attacks, and third-party scripts can silently exfiltrate tokens
  - `HttpOnly` cookies are invisible to JavaScript; only the browser and server can read them
  - **Red flags to search for:** `localStorage.setItem('accessToken'`, `localStorage.getItem('token'`, `sessionStorage.setItem`
  ```typescript
  // ✅ Server route sets the cookie — token never touches client JS
  response.cookies.set('session', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60,  // 1h
    path: '/',
  })

  // ❌ Client code stores the token — XSS readable
  localStorage.setItem('accessToken', token)
  document.cookie = `accessToken=${token}` // still JS-readable without httpOnly
  ```

- [ ] **Cookie set by the server, read by the server** — client components never handle raw tokens, only derived UI state (email, isLoggedIn flag, display name)
- [ ] **Sensitive cookies have `Secure` flag** in production — prevents transmission over HTTP

### 1.2 Session Lifecycle

- [ ] **Token expiry is checked** — read the `exp` claim from the JWT on app load; if expired, call logout or trigger a refresh before any API call fails silently with a 401
- [ ] **Refresh logic exists** — if access tokens expire (common: 1h), there is a code path to exchange `refreshToken` for a new access token without requiring the user to re-login
- [ ] **Logout invalidates server-side session** — not just deletes the client cookie. If using JWTs, add them to a blocklist (Redis) on logout, or use short-lived tokens + refresh rotation
- [ ] **Concurrent session handling defined** — either allow multiple sessions (and list them in account settings) or invalidate old sessions on new login

### 1.3 JWT Handling

- [ ] **JWT signature is verified on every request** — not just decoded (anyone can decode a JWT; only the server can verify it)
- [ ] **Algorithm is pinned** — specify `algorithms: ['RS256']` or `['HS256']` explicitly; never accept `algorithm: 'none'`
- [ ] **`exp`, `iat`, `nbf` claims are validated** — a stolen but expired token must be rejected
- [ ] **JWT is not decoded with `atob()` on the client to get user data** — decode only safe non-sensitive claims (display name, role) and only if the token was already verified server-side; never trust client-decoded claims for auth decisions

### 1.4 Auth Guards

- [ ] **Protected routes guarded in middleware** — not only in the page component (page-level check allows a flash of protected content before redirect)
  ```typescript
  // middleware.ts
  export const config = { matcher: ['/dashboard/:path*', '/api/protected/:path*'] }
  export function middleware(request: NextRequest) {
    const token = request.cookies.get('session')?.value
    if (!token) return NextResponse.redirect(new URL('/login', request.url))
  }
  ```
- [ ] **Every Route Handler re-validates auth independently** — middleware redirects browsers, but API clients bypass middleware; each handler must verify the session itself
- [ ] **Server Actions check auth** — `'use server'` is not a security boundary; anyone can POST to a Server Action endpoint

---

## 2. Input Validation & Injection

### 2.1 Schema Validation

- [ ] **Every Route Handler validates request input with Zod** before touching the database or calling external services:
  ```typescript
  const schema = z.object({
    email: z.string().email(),
    message: z.string().max(2000),
  })
  const result = schema.safeParse(await request.json())
  if (!result.success) return NextResponse.json({ error: 'Invalid input' }, { status: 400 })
  ```
- [ ] **Every Server Action validates input with Zod** — `'use server'` does not sanitize input
- [ ] **Validation happens on the server** — client-side validation is UX only, never the security gate

### 2.2 SQL / NoSQL Injection

- [ ] **Parameterized queries used everywhere** — never string interpolation in SQL:
  ```typescript
  // ✅ Parameterized
  db.query('SELECT * FROM users WHERE id = $1', [userId])

  // ❌ Injection vector
  db.query(`SELECT * FROM users WHERE id = '${userId}'`)
  ```
- [ ] **ORM is used with its built-in query builder** — not raw SQL strings with user input
- [ ] **Search parameters are sanitized** — full-text search inputs are escaped or passed to the search engine via its safe API

### 2.3 XSS (Cross-Site Scripting)

- [ ] **`dangerouslySetInnerHTML` is not used with user-supplied content** — if needed, sanitize with `DOMPurify` first
- [ ] **User-generated content is never rendered as raw HTML** in Server Components via string interpolation in JSX (JSX auto-escapes; the risk is explicit `dangerouslySetInnerHTML`)
- [ ] **Markdown rendered from user input goes through a sanitizer** — libraries like `marked` do not sanitize by default; use `DOMPurify.sanitize(marked.parse(input))`
- [ ] **Content Security Policy (CSP) header is set** — reduces impact of any XSS that slips through:
  ```
  Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'
  ```

### 2.4 Path Traversal & File Upload

- [ ] **File upload MIME type validated server-side** — not only the `Content-Type` header (spoofable); check the actual file signature (magic bytes)
- [ ] **Uploaded files are never served from the same origin as the app** — serve from a CDN or isolated subdomain to prevent stored XSS via uploaded HTML/SVG files
- [ ] **File size limited server-side** — not only on the client; enforce max size in the Route Handler
- [ ] **File names sanitized** — never use the user-supplied filename for storage; generate a UUID key
- [ ] **Presigned URL pattern for S3** — client uploads directly to S3 with a short-lived URL; the backend never proxies binary data

---

## 3. Security Headers

All of these are set in `next.config.ts` `headers()`. Missing headers are the most common finding in web security audits and take ~30 minutes to add.

```typescript
// next.config.ts
async headers() {
  return [{
    source: '/(.*)',
    headers: [
      // Prevent clickjacking (embedding in iframe)
      { key: 'X-Frame-Options', value: 'DENY' },
      // Prevent MIME-type sniffing
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      // Strict HTTPS after first visit (1 year, include subdomains)
      { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
      // Limit referrer leakage
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      // Restrict browser features
      { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
    ],
  }]
}
```

- [ ] `X-Frame-Options: DENY` — clickjacking protection
- [ ] `X-Content-Type-Options: nosniff` — MIME sniffing protection
- [ ] `Strict-Transport-Security` — HSTS (HTTPS enforcement)
- [ ] `Referrer-Policy` — controls what URL is sent in the `Referer` header
- [ ] `Permissions-Policy` — disables browser APIs the app doesn't use
- [ ] **Content Security Policy (CSP)** — the most powerful and most complex header; at minimum set for apps that handle auth or payments:
  ```
  Content-Security-Policy:
    default-src 'self';
    script-src 'self' 'nonce-{SERVER_GENERATED_NONCE}';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https://your-cdn.com;
    connect-src 'self' https://your-api.com;
    frame-ancestors 'none';
    object-src 'none';
    base-uri 'self';
  ```
  Next.js 15 supports nonce-based CSP via middleware. Use the [Next.js CSP guide](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy).

- [ ] **Headers verified with a scanner** — use [securityheaders.com](https://securityheaders.com) or `curl -I https://your-domain.com` against production

---

## 4. Secrets & Environment Variables

### 4.1 Variable Naming

- [ ] **`NEXT_PUBLIC_*` variables contain zero secrets** — they are embedded in the JavaScript bundle and visible to every user who opens DevTools → Sources
- [ ] **Server-only secrets have no `NEXT_PUBLIC_` prefix** — database URLs, API keys, signing secrets, OAuth client secrets
- [ ] **`.env.example` committed** with placeholder values — every variable documented with a one-line comment explaining its purpose
- [ ] **`.env`, `.env.local`, `.env.production` are in `.gitignore`** and never committed

### 4.2 Secret Handling at Runtime

- [ ] **Secrets never logged** — `console.log(process.env)` in any server-side code outputs all secrets to your log aggregator
- [ ] **Secrets never returned in API responses** — strip sensitive fields explicitly before returning; use a DTO or `select` projection
- [ ] **Secrets never passed to the client** — Server Components can access `process.env` but must not pass secret values as props to Client Components
- [ ] **Secrets are validated at startup** with Zod — app crashes immediately on missing config rather than silently failing later:
  ```typescript
  // src/lib/env.ts
  const schema = z.object({
    DATABASE_URL: z.string().url(),
    JWT_SECRET: z.string().min(32),
    NEXT_PUBLIC_APP_URL: z.string().url(),
  })
  export const env = schema.parse(process.env)
  ```

### 4.3 Git History

- [ ] **`git log --all -S 'password\|secret\|token\|key'`** run to check for secrets accidentally committed in history — use `git-secrets` or `truffleHog` for automated scanning
- [ ] **If a secret was committed:** rotate it immediately (it's compromised), then clean the history with `git filter-repo`

---

## 5. API Security

### 5.1 Route Handlers

- [ ] **Every protected Route Handler verifies the session** — not only middleware
- [ ] **Never trust client-supplied user IDs** — read the user identity from the verified session/token, not from `request.body.userId` or query params:
  ```typescript
  // ❌ Attacker can set userId to any value
  const { userId } = await request.json()
  const data = await db.getUserData(userId)

  // ✅ userId comes from the verified session
  const session = await getSession(request)
  if (!session) return unauthorized()
  const data = await db.getUserData(session.userId)
  ```
- [ ] **Authorization checked, not just authentication** — verify the authenticated user has permission to access the specific resource (IDOR — Insecure Direct Object Reference):
  ```typescript
  const listing = await db.getListing(listingId)
  if (listing.ownerId !== session.userId) return forbidden()
  ```
- [ ] **HTTP methods restricted** — a Route Handler that only handles `GET` should reject `POST`/`DELETE` with 405
- [ ] **Response does not expose internal error details** — catch errors and return a generic message; log the real error server-side

### 5.2 Rate Limiting

- [ ] **Auth endpoints rate-limited** — login, register, password reset, OTP verification. Without this: brute force, credential stuffing, account enumeration
- [ ] **AI/LLM endpoints rate-limited** — LLM calls are expensive; an attacker can trigger repeated calls and run up your bill
- [ ] **Rate limiting applied in middleware or at the edge** — not inside the Route Handler (Handler is called after the damage is done):
  ```typescript
  // Using upstash/ratelimit
  const { success } = await ratelimit.limit(ip)
  if (!success) return new NextResponse('Too Many Requests', { status: 429 })
  ```

### 5.3 CORS

- [ ] **CORS not set to `*`** for endpoints that return user data or accept mutations
- [ ] **Allowed origins are explicit** — list of production + staging domains, not a wildcard
- [ ] **Credentials: true** only if cookies are sent cross-origin (rare; usually not needed with same-site cookies)

### 5.4 Server Actions Security

- [ ] **Server Actions are treated as API endpoints** — they accept POST requests from any origin; validate input, check auth, check authorization (same as Route Handlers)
- [ ] **Next.js 14+ built-in CSRF protection for Server Actions is not bypassed** — do not set custom headers that would disable the same-origin check

---

## 6. Dependency Security

- [ ] **`pnpm audit` (or `npm audit`) runs in CI** on every PR — blocks merge on critical/high vulnerabilities
- [ ] **Renovate or Dependabot configured** for automated dependency update PRs
- [ ] **No packages with known abandoned maintenance** — check `npmjs.com` for last publish date and open issues
- [ ] **Minimal third-party dependencies** — every package added is a potential attack vector; prefer native APIs where possible
- [ ] **Lock file committed** (`pnpm-lock.yaml` or `package-lock.json`) — ensures reproducible installs and prevents dependency confusion attacks
- [ ] **`overrides` / `resolutions` used** to patch transitive dependency vulnerabilities when the direct dependency hasn't shipped a fix

---

## 7. Error Handling & Logging

- [ ] **Stack traces never exposed to the client** in production — `error.tsx` shows a user-friendly message, not `error.stack`
- [ ] **Internal implementation details not in error messages** — database error messages, file paths, SQL queries must not reach the response body
- [ ] **Errors logged server-side** with enough context to debug — use a structured logger (Pino, Winston) that sends to a log aggregator, not `console.error`
- [ ] **No PII (Personally Identifiable Information) in logs** — email addresses, phone numbers, passwords, tokens must be redacted or omitted from log lines
- [ ] **`error.tsx` is a Client Component** (required by Next.js) but does not expose `error.message` directly to the user — show a generic message; log `error.message` to the server

---

## 8. GDPR & Data Privacy (EU Projects)

- [ ] **Third-party fonts loaded with `next/font`** not via `<link>` to Google Fonts — Google Fonts requests send the user's IP to Google servers (third-party data processor without consent)
- [ ] **Analytics / tracking scripts require consent** before loading — check if Google Analytics, Hotjar, Meta Pixel are loaded unconditionally
- [ ] **User data deletion implemented** — if users can create accounts, there must be a path to delete all their data (GDPR Article 17)
- [ ] **Data retention policy defined** — logs, backups, user data — how long is it kept, and is it deleted automatically?
- [ ] **Third-party services documented** in a data processing register — what data goes to which vendor (Vercel, AWS, OpenAI, etc.)

---

## Part 2 — AI Agent Readiness

---

## 9. What AI Agents Get Wrong in Security (Common Failure Modes)

AI agents frequently reintroduce security issues because they optimize for "working code" without security context. These are the patterns most often seen without explicit CLAUDE.md rules.

| Failure | Root cause | Fix via |
|---------|-----------|---------|
| Stores auth tokens in `localStorage` | Falls back to SPA tutorial patterns (pre-HttpOnly era) | CLAUDE.md auth rule |
| Sets cookie without `httpOnly` flag | Doesn't know the flag exists; copies minimal examples | CLAUDE.md cookie snippet |
| Skips Zod validation in Route Handler | Assumes `request.json()` is safe | CLAUDE.md: Zod on every handler |
| Trusts `userId` from request body | Doesn't know the IDOR pattern | CLAUDE.md: userId from session only |
| Returns full DB error message to client | Propagates errors naively | CLAUDE.md: generic error messages |
| Adds `NEXT_PUBLIC_` prefix to secrets | Copies env var patterns without understanding naming | CLAUDE.md: env naming rules |
| Skips auth check in Route Handler because middleware exists | Thinks middleware is sufficient | CLAUDE.md: double-check rule |
| Uses `dangerouslySetInnerHTML` with user content | Fastest way to render HTML | CLAUDE.md: sanitize first |
| No rate limiting on auth endpoint | Doesn't think about abuse cases | Skill: create-api-route requires it |
| Logs the full request body | "For debugging" | CLAUDE.md: no PII in logs |
| Sets `Access-Control-Allow-Origin: *` | Default CORS tutorial pattern | CLAUDE.md: explicit origins |
| Forgets `secure: true` on production cookies | Local dev works without it | CLAUDE.md cookie snippet |
| Checks `req.method === 'GET'` but handles `POST` too | Incomplete guard | nextjs-reviewer catches it |
| No authorization check (authn ≠ authz) | Knows user is logged in, forgets ownership check | CLAUDE.md: authz pattern |
| Decodes JWT with `atob()` client-side for access decisions | Convenience over security | CLAUDE.md: JWT rules |

---

## 10. CLAUDE.md Security Snippet

Add this block to the project's `CLAUDE.md` (or `frontend/CLAUDE.md`). Keep it focused — this is hot memory loaded every session.

```markdown
## Security Rules

### Auth tokens
- NEVER store tokens (accessToken, refreshToken, idToken) in localStorage, sessionStorage,
  or via document.cookie from client code.
- Tokens are set exclusively by server-side routes (Route Handlers or Server Actions)
  as HttpOnly cookies:
    response.cookies.set('session', token, { httpOnly: true, secure: true, sameSite: 'lax' })
- Client components receive only derived UI state (email, isLoggedIn) — never raw tokens.
- Always check token expiry (exp claim) on app load. Implement refresh if tokens expire.

### Every Route Handler and Server Action must:
1. Verify the session — read from cookies(), not from request body
2. Check authorization — confirm the authenticated user owns the requested resource
   (never trust userId from request body/params — read from session)
3. Validate input with Zod before any database operation
4. Return generic error messages to the client — log real errors server-side
5. Reject unexpected HTTP methods (return 405)

### Env variables
- NEXT_PUBLIC_ = browser-safe public config only. Zero secrets.
- All secrets validated at startup in src/lib/env.ts (Zod schema).
- Never log process.env or any secret value.

### Secrets in responses
- Strip sensitive fields before returning any object.
- Never expose: passwords, tokens, internal IDs, stack traces, SQL errors, file paths.

### Cookies (when setting from server)
httpOnly: true
secure: process.env.NODE_ENV === 'production'
sameSite: 'lax'
maxAge: <explicit, not 'session'>
path: '/'

### Security headers
- next.config.ts must have a headers() function.
- Minimum required: X-Frame-Options, X-Content-Type-Options, Referrer-Policy,
  Strict-Transport-Security, Permissions-Policy.
- Do not add headers only to individual routes — apply to '/(.*)'

### Input
- Never use dangerouslySetInnerHTML with user-supplied content without DOMPurify sanitization.
- Never build SQL strings with user input — use parameterized queries or ORM.

### Rate limiting
- Auth endpoints (/api/auth/*) must have rate limiting before the handler logic.
- AI/LLM-triggering endpoints must have rate limiting.
```

---

## 11. Skill: `security-review`

Create at `.claude/skills/security-review.md`:

```markdown
# Skill: Security Review

When asked to review a file, feature, or PR for security:

## Checklist (check in this order)

### 1. Auth & Token Storage
- Any localStorage / sessionStorage writes with token/auth content?
- Any document.cookie writes from client code?
- HttpOnly + Secure + SameSite flags on all auth cookies?
- Token expiry checked? Refresh logic present?

### 2. Route Handlers & Server Actions
- Session verified at the top of every handler?
- Authorization checked (not just authentication)?
- userId read from session, NOT from request body/params?
- Input validated with Zod before any side effects?
- Error messages generic to client, detailed to server logs?

### 3. Env Variables & Secrets
- Any secrets in NEXT_PUBLIC_ variables?
- Any secrets in console.log / logger calls?
- Any secrets returned in API responses?

### 4. Security Headers
- headers() function present in next.config.ts?
- X-Frame-Options, X-Content-Type-Options, HSTS present?
- CSP configured for apps handling auth or payments?

### 5. Input Handling
- dangerouslySetInnerHTML with user content? (require DOMPurify)
- String interpolation in SQL? (require parameterized queries)
- File uploads: MIME validated server-side? Size limited? UUID filename?

### 6. Dependencies
- Any packages with known CVEs? (check via pnpm audit)
- Any package not updated in >2 years on a critical path?

## Output format
For each finding:
  Severity: Critical / High / Medium / Low
  File: path/to/file.ts:line
  Issue: one-line description
  Risk: what can an attacker do?
  Fix: minimal code change or pattern

Do NOT modify any files. Report only.
```

---

## 12. Subagent: `security-reviewer`

Create at `.claude/agents/security-reviewer.md`:

```markdown
---
name: security-reviewer
description: Reviews Next.js code for auth vulnerabilities, injection risks, secret leaks, missing security headers, and OWASP Top 10 patterns. Use when adding auth code, Route Handlers, Server Actions, or file upload. Also use before any production deploy.
tools: Read, Glob, Grep
model: sonnet
---

You are a security-focused code reviewer specializing in Next.js and Node.js.
You look for vulnerabilities, not style issues.

## Review order

1. **Search for token storage red flags**
   Grep for: localStorage.setItem, sessionStorage.setItem, document.cookie
   Flag any that involve token / auth / session / key / secret

2. **Read every Route Handler and Server Action**
   For each: Is the session verified? Is authorization checked? Is userId from the session?
   Is input validated with Zod? Are errors generic to client?

3. **Check next.config.ts for headers()**
   List which headers are present and which are missing.

4. **Grep for env variable misuse**
   Search for NEXT_PUBLIC_ followed by SECRET, KEY, PASSWORD, TOKEN.
   Search for console.log patterns near env vars.

5. **Check for injection patterns**
   Grep for template literals containing SQL keywords (SELECT, INSERT, UPDATE, WHERE).
   Grep for dangerouslySetInnerHTML.

6. **Check rate limiting**
   Grep for /api/auth/ route files — is there a ratelimit call before the handler logic?

## Report format

```
## Security Review Report

### Critical
- file:line — issue — risk — fix

### High
- file:line — issue — risk — fix

### Medium / Low
- (grouped)

### Not Found (checked but clean)
- List of areas checked with no findings
```

Do not modify any files.
```

---

## 13. Hook: Block Writes to Auth-Related Files

Add to `.claude/settings.json` — warns before editing sensitive files:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'echo \"$CLAUDE_FILE_PATH\" | grep -qiE \"auth|session|cookie|jwt|token|middleware\" && echo \"⚠️  SECURITY-SENSITIVE FILE — verify auth rules and HttpOnly cookie pattern before saving\" >&2 || true'"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "comment": "Grep for localStorage token writes after any file edit",
            "command": "bash -c 'git diff --unified=0 | grep \"+.*localStorage.*[Tt]oken\\|+.*sessionStorage.*[Tt]oken\\|+.*document\\.cookie\" && echo \"🔴 WARNING: Token written to client storage — use HttpOnly cookie from server route\" >&2 || true'"
          }
        ]
      }
    ]
  }
}
```

---

## 14. Quick Audit Checklist (Yes/No)

Ask Claude: *"Answer Yes/No for each item based on the current codebase."*

**Auth & Sessions**
- [ ] Auth tokens in `HttpOnly` cookies (not `localStorage` / `sessionStorage`)?
- [ ] Cookies have `Secure` flag in production?
- [ ] Cookies have `SameSite: lax` or `strict`?
- [ ] Token expiry (`exp` claim) checked on app load?
- [ ] Refresh logic implemented for expiring access tokens?
- [ ] Logout invalidates the server-side session / adds token to blocklist?
- [ ] Auth guard in `middleware.ts` with explicit `matcher`?

**Route Handlers & Server Actions**
- [ ] Every protected handler verifies session independently (not only middleware)?
- [ ] Every handler reads `userId` from session, not from request body?
- [ ] Authorization checked — user owns the resource before read/write?
- [ ] Every handler validates input with Zod?
- [ ] Client receives generic error messages (not stack traces / SQL errors)?
- [ ] HTTP methods restricted per handler?

**Security Headers**
- [ ] `X-Frame-Options` set?
- [ ] `X-Content-Type-Options` set?
- [ ] `Strict-Transport-Security` set?
- [ ] `Referrer-Policy` set?
- [ ] `Permissions-Policy` set?
- [ ] CSP configured (for apps handling auth / payments)?

**Secrets**
- [ ] Zero secrets in `NEXT_PUBLIC_*` variables?
- [ ] Secrets validated at startup with Zod?
- [ ] No `console.log` of secrets or full `process.env`?
- [ ] Secrets stripped from API responses?
- [ ] No secrets in git history (`git log --all -S 'secret'`)?

**Input & Injection**
- [ ] `dangerouslySetInnerHTML` with user content always sanitized with DOMPurify?
- [ ] No string interpolation in SQL queries?
- [ ] File uploads: server-side MIME validation, size limit, UUID filename?

**Rate Limiting**
- [ ] Auth endpoints (`/api/auth/*`) rate-limited?
- [ ] AI/LLM-triggering endpoints rate-limited?

**Dependencies**
- [ ] `pnpm audit` runs in CI and blocks on critical/high?
- [ ] Renovate or Dependabot configured?

**GDPR (EU projects)**
- [ ] No Google Fonts via `<link>` (use `next/font` to avoid data transfer)?
- [ ] Analytics/tracking requires consent before loading?
- [ ] User data deletion endpoint implemented?

**AI Readiness**
- [ ] Security rules in `CLAUDE.md` (auth, env vars, Zod, error messages)?
- [ ] `.claude/skills/security-review.md` exists?
- [ ] `security-reviewer` agent exists?
- [ ] Hook warns on edits to auth-related files?

---

## 15. Scoring

Count Yes answers (max 40):

| Score | Assessment |
|-------|-----------|
| 35–40 | Strong security posture — safe to scale and handle sensitive user data |
| 27–34 | Acceptable baseline — address Critical and High gaps before launch |
| 16–26 | Significant gaps — auth or header issues likely present; fix before handling payments or PII |
| <16 | High risk — do not launch until auth token storage, headers, and input validation are fixed |

> **Note:** Score <27 with any Critical item (localStorage tokens, no input validation, secrets in NEXT_PUBLIC_) = do not launch regardless of total score.
