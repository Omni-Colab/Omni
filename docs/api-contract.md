# SuperApp API Contract: Identity & Authentication (v0.2)

> **Task:** SUP-00
> **Status:** Draft, pending review by both developers
> **Rule:** This file is the single source of truth for the HTTP contract. Backend and frontend must not deviate from it. Changes go through a PR that both developers approve.

---

## 0. v0.1 foundation: health checks

These endpoints are part of the v0.1 foundation and are intentionally outside the `/api/v1/auth` namespace.

### `GET /healthz`

Liveness endpoint. Returns `200` when the service process is alive.

| Status | Body |
|---|---|
| 200 | `{ "status": "ok" }` |

### `GET /readyz`

Readiness endpoint. Returns `200` only when required backing services are reachable.

| Status | Body |
|---|---|
| 200 | `{ "status": "ready", "checks": { "postgres": "ok", "redis": "ok", "nats": "ok" } }` |
| 503 | `{ "status": "not_ready", "checks": { "postgres": "down", "redis": "ok", "nats": "ok" } }` |

The ready check must report Postgres, Redis, and NATS connectivity; v0.1 tests rely on this signal.

---

## 1. Conventions

| Item | Decision |
|---|---|
| Auth base path | `/api/v1/auth` |
| User profile base path | `/api/v1/users` |
| Format | JSON request and response (`Content-Type: application/json`), except `204` responses with no body |
| IDs | UUID, lowercase string |
| Timestamps | RFC 3339, UTC (`2026-10-25T12:00:00Z`) |
| Field naming | `snake_case` |
| Email handling | Trimmed and lowercased before validation and storage; unique |
| Auth header (protected routes) | `Authorization: Bearer <access_token>` |
| Refresh cookie path | `/api/v1/auth` |
| CORS | Allowed origin from env (`CORS_ALLOWED_ORIGIN`), `Access-Control-Allow-Credentials: true` |
| Frontend fetch | Calls that rely on the refresh cookie must use `credentials: "include"` |

---

## 2. Error format

Every non-2xx response uses this body:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "details": [
      { "field": "password", "message": "Must be at least 8 characters." }
    ]
  }
}
```

`details` is present only for `VALIDATION_ERROR`; omit it otherwise.

| HTTP | `code` | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Malformed JSON or a field fails validation |
| 400 | `TOKEN_INVALID_OR_EXPIRED` | Verification or reset token is unknown, expired, or already used |
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 401 | `INVALID_REFRESH_TOKEN` | Refresh token missing, unknown, expired, revoked, or reused |
| 401 | `UNAUTHORIZED` | Missing, invalid, or expired access token on a protected route |
| 403 | `EMAIL_NOT_VERIFIED` | Correct credentials but the email is not verified |
| 409 | `EMAIL_ALREADY_REGISTERED` | Registration with an existing email |
| 429 | `RATE_LIMITED` | Rate limit exceeded; response includes `Retry-After` (seconds) |
| 500 | `INTERNAL_ERROR` | Unexpected failure; never leak internals |

---

## 3. Validation rules

| Field | Rule |
|---|---|
| `email` | Required, valid format, max 254 chars |
| `password` | Required, 8 to 128 characters, no composition rules (length matters more) |
| `current_password` | Required, 8 to 128 characters |
| `new_password` | Required, 8 to 128 characters |
| `display_name` | Required on register and profile update, 1 to 50 characters after trimming |
| `device_name` | Optional, 1 to 100 characters after trimming |
| `token` | Required, non-empty string, max 256 chars |

---

## 4. Shared objects

**User**

```json
{
  "id": "0b7c1a52-3d0e-4c0a-9f2e-6a1d6f3b9e11",
  "email": "alex@example.com",
  "display_name": "Alex",
  "email_verified": true,
  "created_at": "2026-10-25T12:00:00Z"
}
```

**AuthResponse** (returned by login and refresh)

```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": { "...User object..." }
}
```

`expires_in` is in seconds. The refresh token is **never** in the body; it travels only in the cookie (section 6).

---

## 5. Endpoints

### 5.1 `POST /api/v1/auth/register`

Creates an account and sends a verification email. Does **not** log the user in.

Request:

```json
{ "email": "alex@example.com", "password": "correct horse battery", "display_name": "Alex" }
```

| Status | Body |
|---|---|
| 201 | `{ "user": <User> }` (`email_verified` is `false`) |
| 400 | `VALIDATION_ERROR` |
| 409 | `EMAIL_ALREADY_REGISTERED` |
| 429 | `RATE_LIMITED` |

### 5.2 `POST /api/v1/auth/verify-email`

Request: `{ "token": "<token from email>" }`

| Status | Body |
|---|---|
| 200 | `{ "message": "Email verified." }` |
| 400 | `VALIDATION_ERROR` or `TOKEN_INVALID_OR_EXPIRED` |
| 429 | `RATE_LIMITED` |

### 5.3 `POST /api/v1/auth/resend-verification`

Resends the email verification link for a user whose account exists but is not yet verified.

Request: `{ "email": "alex@example.com" }`

Always returns the same response regardless of whether the account exists, to avoid account enumeration. If the user exists and is unverified, a new verification email is sent.

| Status | Body |
|---|---|
| 202 | `{ "message": "If an account exists for that email and is unverified, a new verification link has been sent." }` |
| 400 | `VALIDATION_ERROR` |
| 429 | `RATE_LIMITED` |

### 5.4 `POST /api/v1/auth/login`

Request:

```json
{ "email": "alex@example.com", "password": "correct horse battery", "device_name": "Chrome on Ubuntu" }
```

`device_name` is optional (max 100 chars). Order of checks: validate input, find user and verify password, then check `email_verified`. This keeps `EMAIL_NOT_VERIFIED` from leaking account existence.

| Status | Body |
|---|---|
| 200 | `<AuthResponse>` + `Set-Cookie` refresh token |
| 400 | `VALIDATION_ERROR` |
| 401 | `INVALID_CREDENTIALS` |
| 403 | `EMAIL_NOT_VERIFIED` |
| 429 | `RATE_LIMITED` |

### 5.5 `POST /api/v1/auth/refresh`

No body. Reads the refresh cookie, rotates it, and returns a new access token. The frontend calls this on page load to restore the session, since the access token only lives in memory.

| Status | Body |
|---|---|
| 200 | `<AuthResponse>` + new `Set-Cookie` refresh token |
| 401 | `INVALID_REFRESH_TOKEN` + cookie cleared |
| 429 | `RATE_LIMITED` |

### 5.6 `POST /api/v1/auth/logout`

No body. Reads the refresh cookie only, so it works even when the access token has expired. Idempotent: always returns `204` and clears the cookie, even if the cookie was missing or invalid.

| Status | Body |
|---|---|
| 204 | none, cookie cleared |

### 5.7 `POST /api/v1/auth/forgot-password`

Request: `{ "email": "alex@example.com" }`

Always returns the same response, whether or not the email exists, to prevent account enumeration. If the account exists and is verified, a reset email is sent.

| Status | Body |
|---|---|
| 202 | `{ "message": "If an account exists for that email, a reset link has been sent." }` |
| 400 | `VALIDATION_ERROR` (malformed email only) |
| 429 | `RATE_LIMITED` |

### 5.8 `POST /api/v1/auth/reset-password`

Request: `{ "token": "<token from email>", "new_password": "new strong password" }`

On success the token is consumed, the password is re-hashed with Argon2id, and **all sessions of that user are revoked**.

| Status | Body |
|---|---|
| 200 | `{ "message": "Password updated." }` |
| 400 | `VALIDATION_ERROR` or `TOKEN_INVALID_OR_EXPIRED` |
| 429 | `RATE_LIMITED` |

### 5.9 `GET /api/v1/users/me`

Protected route. Returns the authenticated user's profile. This is the first API route that exercises the JWT authentication middleware in a real endpoint.

| Status | Body |
|---|---|
| 200 | `{ "user": <User> }` |
| 401 | `UNAUTHORIZED` |
| 429 | `RATE_LIMITED` |

### 5.10 `PATCH /api/v1/users/me`

Protected route. Updates the authenticated user's `display_name`.

Request:

```json
{ "display_name": "Alex Morgan" }
```

| Status | Body |
|---|---|
| 200 | `{ "user": <User> }` |
| 400 | `VALIDATION_ERROR` |
| 401 | `UNAUTHORIZED` |
| 429 | `RATE_LIMITED` |

### 5.11 `POST /api/v1/users/me/change-password`

Protected route. Validates the current password, then rehashes the password and revokes all sessions for the user except the current one until the next refresh or re-authentication cycle.

Request:

```json
{ "current_password": "old-password", "new_password": "new-strong-password" }
```

| Status | Body |
|---|---|
| 200 | `{ "message": "Password updated." }` |
| 400 | `VALIDATION_ERROR` |
| 401 | `INVALID_CREDENTIALS` or `UNAUTHORIZED` |
| 429 | `RATE_LIMITED` |

---

## 6. Token and session design

### 6.1 Access token

| Item | Decision |
|---|---|
| Format | JWT, signed HS256 with `JWT_SECRET` (min 32 bytes, from env) |
| Expiry | 15 minutes |
| Claims | `iss` = `superapp`, `sub` = user ID, `sid` = session ID, `iat`, `exp` |
| Frontend storage | In memory only. Never `localStorage` or `sessionStorage` |
| Revocation | Stateless. After logout it stays valid until `exp` (max 15 min). Known limitation |

### 6.2 Refresh token

| Item | Decision |
|---|---|
| Format | Opaque random value, 32 bytes, base64url |
| Server storage | SHA-256 hash only; the raw value is never stored |
| Lifetime | 30 days, fixed from login (no sliding extension) |
| Rotation | New refresh token on every `/refresh`; the old hash is kept as `prev_refresh_token_hash` |
| Reuse detection | If a request presents the previous token, the whole session is revoked and `INVALID_REFRESH_TOKEN` is returned |

### 6.3 Cookie

| Attribute | Value |
|---|---|
| Name | `refresh_token` |
| `HttpOnly` | yes |
| `Secure` | yes in staging and production; off only for local HTTP development (`COOKIE_SECURE=false`) |
| `SameSite` | `Lax` by default; configurable via `COOKIE_SAMESITE` (see open decision 1) |
| `Path` | `/api/v1/auth` |
| `Max-Age` | `2592000` (30 days) |
| Clearing | Same name, `Path`, `Secure`, `SameSite`, with `Max-Age=0` |

### 6.4 One-time email tokens

| Token | Format | Expiry | Rules |
|---|---|---|---|
| Email verification | 32 random bytes, base64url, stored as SHA-256 hash | 24 hours | One use. A new token invalidates older ones for the same user |
| Password reset | Same | 1 hour | One use. A new token invalidates older ones for the same user |

Email links point to the frontend, which then calls the API:
`{FRONTEND_BASE_URL}/verify-email?token=...` and `{FRONTEND_BASE_URL}/reset-password?token=...`

### 6.5 Passwords

| Item | Decision |
|---|---|
| Algorithm | Argon2id |
| Parameters | memory 64 MiB, iterations 3, parallelism 2, salt 16 bytes, key 32 bytes |
| Stored format | PHC string (`$argon2id$v=19$m=65536,t=3,p=2$...`), so parameters can change later |
| Max input length | 128 characters (bounds hashing cost) |

---

## 7. Rate limits

Phase 1 runs a single instance, so limits are in memory behind the `RateLimiter` interface. Redis replaces the implementation in v0.4.

| Endpoint | Limit |
|---|---|
| `register` | 5 per hour per IP |
| `login` | 10 per 15 min per IP, and 5 per 15 min per email |
| `resend-verification` | 3 per hour per email, and 10 per hour per IP |
| `forgot-password` | 3 per hour per email, and 10 per hour per IP |
| `verify-email` | 10 per hour per IP |
| `reset-password` | 10 per hour per IP |
| `change-password` | 10 per hour per user |
| `refresh` | 60 per minute per IP |

Exceeded limits return `429`, `RATE_LIMITED`, and a `Retry-After` header.

---

## 8. Go interfaces

Package: `backend/internal/modules/identity`. Other modules depend on these interfaces, never on identity tables.

```go
type PasswordHasher interface {
    Hash(password string) (string, error)
    Verify(password, hash string) bool
}

type TokenService interface {
    IssueAccessToken(userID, sessionID uuid.UUID) (token string, expiresAt time.Time, err error)
    ParseAccessToken(token string) (*AccessClaims, error) // validates signature and expiry
    NewOpaqueToken() (raw, hash string, err error)        // refresh, verification, reset
    HashToken(raw string) string                          // SHA-256 hex
}

type UserRepository interface {
    Create(ctx context.Context, u *User) error // returns ErrEmailTaken on duplicate
    GetByID(ctx context.Context, id uuid.UUID) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    MarkEmailVerified(ctx context.Context, id uuid.UUID) error
    UpdatePasswordHash(ctx context.Context, id uuid.UUID, hash string) error
    UpdateDisplayName(ctx context.Context, id uuid.UUID, displayName string) error
}

type SessionRepository interface {
    Create(ctx context.Context, s *Session) error
    GetByRefreshHash(ctx context.Context, hash string) (s *Session, isPrevious bool, err error)
    ListByUser(ctx context.Context, userID uuid.UUID) ([]*Session, error)
    RotateRefresh(ctx context.Context, id uuid.UUID, newHash string, newExpiry time.Time) error
    Revoke(ctx context.Context, id uuid.UUID) error
    RevokeAllForUser(ctx context.Context, userID uuid.UUID) error
}

type OneTimeTokenRepository interface {
    Create(ctx context.Context, t *OneTimeToken) error // invalidates older tokens of the same purpose
    Consume(ctx context.Context, purpose, hash string) (userID uuid.UUID, err error) // atomic: valid, unexpired, unused
}

type EmailSender interface {
    SendVerificationEmail(ctx context.Context, to, displayName, link string) error
    SendPasswordResetEmail(ctx context.Context, to, displayName, link string) error
}

type RateLimiter interface {
    Allow(ctx context.Context, key string, limit int, window time.Duration) (allowed bool, retryAfter time.Duration, err error)
}
```

`Session` fields: `id`, `user_id`, `refresh_token_hash`, `prev_refresh_token_hash`, `device_name`, `user_agent`, `ip`, `created_at`, `expires_at`, `revoked_at`, `last_used_at`.

Interfaces beyond the five required ones (`OneTimeTokenRepository`, `RateLimiter`) are included here so SUP-07, 08, 09 and 10 do not each invent their own.

---

## 9. Data ownership

| Item | Decision |
|---|---|
| Owned tables | `users`, `sessions`, `verification_tokens`, `password_reset_tokens` |
| Owner | SUP-06 (Alex) |
| Rule | Only the owner changes identity models and migrations. The other developer coordinates through a PR comment or message first |
| Migrations | Plain SQL files in `backend/migrations/`, numbered sequentially |

---

## 10. Open decisions (resolve in the SUP-00 PR review)

| # | Question | Default if no objection |
|---|---|---|
| 1 | **Cookie `SameSite` in staging:** if the frontend and API are on different registrable domains, `Lax` will not send the cookie and staging needs `None` + `Secure`. Decide when picking hosting in staging. | `Lax` in local/dev; `None` + `Secure` only when cross-site cookies are required |
| 2 | **Duplicate registration:** `409 EMAIL_ALREADY_REGISTERED` is clearer for users but reveals that an email exists. Rate limiting reduces the abuse. | Keep `409` |

---

## Appendix: v0.2 scope notes

This contract intentionally includes the user profile flow required to validate authenticated access control and account-management behavior. The session list/revoke endpoints remain out of scope for v0.2 and should be added in a later contract unless the team explicitly decides to include them in this PR.
