# VibeMail Engine — Build Contract

Status: **Draft v1** · Owner: Terry Kim · Date: 2026-10-25

This document is the single source of truth for the VibeMail engine build. Anything not specified here is out of scope. Changes require an amendment to this file, merged before the code that depends on them.

---

## 0. Fixed decisions

| Area | Decision |
|---|---|
| Language / runtime | TypeScript (strict), Node.js 24 LTS |
| HTTP framework | Hono (Express acceptable only if Hono is blocked; the contract is framework-neutral at the HTTP layer) |
| Store | PostgreSQL on Supabase, accessed only through repository interfaces (§5) |
| Client auth | Engine-issued JWT, sent as `Authorization: Bearer <token>` |
| Google OAuth scopes | `openid`, `email`, `https://www.googleapis.com/auth/gmail.modify` |
| Gmail read depth | `users.messages.get` with `format=full`; bodies decoded, attachment **metadata** only |
| List source of truth | Gmail, live on every call; results upserted into the store before responding |
| Error format | Custom envelope with a closed code union (§3) |
| Tests | Vitest; Gmail mocked at HTTP layer with `msw` + recorded fixtures; store mocked with in-memory fakes; zero real network calls |
| Build | Two sessions; server first, schema second (§6) |

---

## 1. Acceptance criteria

The project is **complete** when every item below is true. Each is verifiable by a command or a named test.

### Build and quality
- **AC-1** `npm run typecheck` (`tsc --noEmit`, `strict: true`) exits 0.
- **AC-2** `npm run lint` exits 0.
- **AC-3** `npm test` (Vitest) exits 0 with no skipped or `.only` tests, and makes zero real outbound network calls (`msw` configured with `onUnhandledRequest: "error"`).
- **AC-4** No secret is committed. All configuration is read from env vars listed in `.env.example`: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `JWT_SECRET`, `JWT_TTL_SECONDS`, `TOKEN_ENCRYPTION_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`.

### Endpoint behavior (each has at least one passing route-level test)
- **AC-5** OAuth callback exchanges a valid `code`, creates or updates the account, stores the refresh token encrypted, and returns a JWT that authenticates the other three endpoints.
- **AC-6** Every error case listed in §2 for every endpoint has a test that asserts **both** the HTTP status **and** the `error.code`.
- **AC-7** Message list returns messages in Gmail order, supports `limit`, `pageToken`, `q`, and `unread`, and every returned message is present in the store afterward with fields matching §4.
- **AC-8** Message send produces a valid RFC 5322 message (asserted by decoding the `raw` sent to the Gmail mock): correct `To`/`Cc`/`Bcc`/`Subject`, `multipart/alternative` when `html` is provided, and `In-Reply-To`/`References`/`threadId` when `replyToMessageId` is provided. The sent message is upserted and returned.
- **AC-9** Mark-as-read/unread is idempotent: calling it twice with the same value returns 200 both times and leaves Gmail labels and the stored `is_unread` consistent.
- **AC-10** Any response body never contains a Google access token, refresh token, or `TOKEN_ENCRYPTION_KEY`-derived material.
- **AC-11** An expired Google access token is refreshed transparently; a revoked refresh token (`invalid_grant`) yields `401 GMAIL_REAUTH_REQUIRED`.

### Data
- **AC-12** The body-extraction function is unit-tested against fixtures for: plain-only, HTML-only, `multipart/alternative`, `multipart/mixed` with attachment, nested multipart, and a non-UTF-8 charset.
- **AC-13** The Supabase migration (Session 2) creates exactly the tables and columns in §4, and the Supabase repository implementation passes the **same** repository contract test suite that the in-memory fake passes.

### Process
- **AC-14** The sequencing rule in §6 was followed: the schema PR's review and merge both post-date a green CI run on the server PR's merge commit.

---

## 2. API endpoint contracts

All endpoints return `application/json`. Authenticated endpoints require `Authorization: Bearer <jwt>`. Field types are TypeScript.

### Shared types

```ts
type ISODateTime = string;          // RFC 3339, UTC, e.g. "2026-10-25T14:03:00.000Z"

interface EmailAddress {
  name: string | null;              // display name, null if absent
  email: string;                    // lowercased address
}

interface AttachmentMeta {
  partId: string;
  filename: string;
  mimeType: string;
  size: number;                     // bytes
  attachmentId: string;             // Gmail attachment id (bytes not stored)
}

// Response DTO for a message. Mirrors §4 minus internal columns.
interface Message {
  id: string;                       // Gmail message id
  threadId: string;
  labelIds: string[];
  isUnread: boolean;
  snippet: string;
  subject: string | null;
  from: EmailAddress | null;
  to: EmailAddress[];
  cc: EmailAddress[];
  bcc: EmailAddress[];
  replyTo: EmailAddress[];
  date: ISODateTime | null;         // parsed Date header
  internalDate: ISODateTime;        // Gmail receive time
  rfc822MessageId: string | null;
  inReplyTo: string | null;
  references: string[];
  bodyText: string | null;
  bodyHtml: string | null;
  attachments: AttachmentMeta[];
  historyId: string;
  sizeEstimate: number;
}
```

### Errors common to all authenticated endpoints

| HTTP | `error.code` | When |
|---|---|---|
| 401 | `AUTH_MISSING` | No `Authorization` header, or not `Bearer` |
| 401 | `AUTH_INVALID` | JWT signature invalid, malformed, or `sub` does not match an account |
| 401 | `AUTH_EXPIRED` | JWT `exp` is in the past |
| 401 | `GMAIL_REAUTH_REQUIRED` | Google refresh returned `invalid_grant` (revoked or expired grant) |
| 403 | `GMAIL_SCOPE_INSUFFICIENT` | Gmail returns 403 `insufficientPermissions` |
| 429 | `GMAIL_RATE_LIMITED` | Gmail returns 429, or 403 with reason `rateLimitExceeded` / `userRateLimitExceeded`. `details.retryAfterSeconds` set when known |
| 502 | `GMAIL_UNAVAILABLE` | Gmail returns 5xx, times out (10 s), or a network error occurs |
| 503 | `STORE_UNAVAILABLE` | Repository call fails |
| 500 | `INTERNAL` | Any unhandled error. The message is generic and includes no stack trace |

---

### 2.1 OAuth callback

`GET /auth/google/callback`

Google redirects the browser here; the endpoint returns JSON (no redirect).

> **Supporting route (not one of the four core endpoints):** `GET /auth/google` 302-redirects to Google's consent URL with `access_type=offline`, `prompt=consent`, the scopes in §0, and a `state` value. `state` is a stateless HMAC-signed token (`JWT_SECRET`) containing a random nonce and a 10-minute expiry.

**Request** (query string)

```ts
interface OAuthCallbackQuery {
  code?: string;                    // present on success
  state: string;                    // required
  error?: string;                   // present when user denied, e.g. "access_denied"
}
```

**Behavior**
1. Verify `state` signature and expiry.
2. Exchange `code` at Google's token endpoint.
3. Verify the `id_token`; take `sub` and `email`.
4. Confirm the granted scopes include `gmail.modify`.
5. Upsert the account by `google_sub`. Store the refresh token encrypted. If Google returned no refresh token and the account has none stored, fail.
6. Issue JWT: HS256, `sub = account.id`, `iat`, `exp = iat + JWT_TTL_SECONDS` (default 86400).

**Response** `200 OK`

```ts
interface OAuthCallbackResponse {
  token: string;                    // engine JWT
  tokenType: "Bearer";
  expiresAt: ISODateTime;
  account: { id: string; email: string };
}
```

**Errors**

| HTTP | `error.code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `state` missing, or both `code` and `error` missing |
| 400 | `OAUTH_STATE_INVALID` | `state` signature bad or expired |
| 400 | `OAUTH_ACCESS_DENIED` | `error` query param present (user declined consent) |
| 400 | `OAUTH_CODE_INVALID` | Google token endpoint returns `invalid_grant` for the code |
| 400 | `OAUTH_NO_REFRESH_TOKEN` | No refresh token returned and none stored for this account |
| 403 | `GMAIL_SCOPE_INSUFFICIENT` | Granted scopes lack `gmail.modify` |
| 502 | `OAUTH_EXCHANGE_FAILED` | Token endpoint 5xx, timeout, or malformed response |
| 503 | `STORE_UNAVAILABLE` | Account upsert fails |

---

### 2.2 Message list

`GET /messages`

**Request** (query string)

```ts
interface ListMessagesQuery {
  limit?: number;                   // integer 1–100, default 20 → Gmail maxResults
  pageToken?: string;               // opaque, from a previous nextPageToken → Gmail pageToken
  q?: string;                       // Gmail search syntax, max 500 chars → Gmail q
  unread?: "true" | "false";        // "true" adds labelIds=UNREAD to the Gmail call
}
```

**Behavior**
1. Call `users.messages.list` with the mapped params and `includeSpamTrash=false`. There is no default label filter.
2. For each returned id, call `users.messages.get?format=full`, with concurrency of at most 10.
3. Map each message per §4 and upsert all of them in one repository call.
4. Respond in Gmail's order. If the upsert fails, the request fails with `503`. Data is never returned unpersisted.

**Response** `200 OK`

```ts
interface ListMessagesResponse {
  messages: Message[];
  nextPageToken: string | null;     // Gmail nextPageToken, null when absent
  resultSizeEstimate: number;       // Gmail resultSizeEstimate, 0 when absent
}
```

**Errors** (in addition to common)

| HTTP | `error.code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | `limit` not an integer in 1–100; `q` > 500 chars; `unread` not `"true"`/`"false"`; unknown query param |
| 400 | `INVALID_PAGE_TOKEN` | Gmail rejects `pageToken` with 400 |

A message deleted between `list` and `get` (Gmail 404) is skipped silently and is not an error.

---

### 2.3 Message send

`POST /messages/send`

**Request** (JSON body)

```ts
interface SendMessageRequest {
  to: string[];                     // ≥1 address
  cc?: string[];
  bcc?: string[];
  subject: string;                  // 1–998 chars
  text: string;                     // plain-text body, required
  html?: string;                    // optional; triggers multipart/alternative
  replyToMessageId?: string;        // Gmail message id in this account
}
```

Validation: every address must be a valid RFC 5322 addr-spec. Total recipients across `to` + `cc` + `bcc` must be ≤ 100. No CR or LF in `subject` or any address (header-injection guard). Max body size 1 MB.

**Behavior**
1. If `replyToMessageId` is set, fetch it (`format=metadata`, headers `Message-ID`, `References`). Set `In-Reply-To = <its Message-ID>` and `References = <its References> <its Message-ID>`, and pass its `threadId`. The subject is used exactly as given.
2. Build MIME. `From` is the account email. With `html`, the message is `multipart/alternative` (text first, then html). UTF-8 throughout, with RFC 2047 encoding for non-ASCII headers.
3. Call `users.messages.send` with `{ raw: base64url(mime), threadId? }`.
4. Fetch the sent message (`format=full`), upsert it, and return it.

**Response** `201 Created`

```ts
interface SendMessageResponse {
  message: Message;
}
```

**Errors** (in addition to common)

| HTTP | `error.code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Any validation rule above fails; `details.fields` lists offending paths |
| 404 | `REPLY_TARGET_NOT_FOUND` | `replyToMessageId` returns Gmail 404 |
| 413 | `PAYLOAD_TOO_LARGE` | Body > 1 MB |
| 422 | `GMAIL_SEND_REJECTED` | Gmail returns 400 on `send` (for example, an invalid recipient per Gmail) |

If the send succeeds but the post-send fetch or upsert fails, respond `503 STORE_UNAVAILABLE` (or the matching Gmail code) with `details.sentMessageId` set. The email **was** sent, and clients must not retry the send.

---

### 2.4 Mark as read / unread

`PATCH /messages/:id`

**Request**

```ts
// path
interface MarkReadParams { id: string }     // Gmail message id
// body
interface MarkReadRequest { read: boolean }
```

**Behavior**
1. Call `users.messages.modify` with `removeLabelIds: ["UNREAD"]` when `read: true`, or `addLabelIds: ["UNREAD"]` when `read: false`.
2. Map the returned labels, update the stored row's `label_ids` and `is_unread` (upsert the full message via `get?format=full` if the row doesn't exist), and return it.
3. The call is idempotent: repeating the same value returns 200 with identical state.

**Response** `200 OK`

```ts
interface MarkReadResponse {
  message: Message;
}
```

**Errors** (in addition to common)

| HTTP | `error.code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Body missing `read`, `read` not boolean, extra fields, or empty `id` |
| 404 | `MESSAGE_NOT_FOUND` | Gmail `modify` returns 404 |

If Gmail succeeds but the store fails, the response is `503`. A client retry is safe because the call is idempotent.

---

## 3. Error envelope

Every non-2xx response has exactly this shape:

```ts
type ErrorCode =
  | "VALIDATION_FAILED"
  | "AUTH_MISSING" | "AUTH_INVALID" | "AUTH_EXPIRED"
  | "OAUTH_STATE_INVALID" | "OAUTH_ACCESS_DENIED" | "OAUTH_CODE_INVALID"
  | "OAUTH_NO_REFRESH_TOKEN" | "OAUTH_EXCHANGE_FAILED"
  | "GMAIL_REAUTH_REQUIRED" | "GMAIL_SCOPE_INSUFFICIENT" | "GMAIL_RATE_LIMITED"
  | "GMAIL_UNAVAILABLE" | "GMAIL_SEND_REJECTED"
  | "INVALID_PAGE_TOKEN" | "MESSAGE_NOT_FOUND" | "REPLY_TARGET_NOT_FOUND"
  | "PAYLOAD_TOO_LARGE"
  | "STORE_UNAVAILABLE"
  | "INTERNAL";

interface ErrorResponse {
  error: {
    code: ErrorCode;
    message: string;                // human-readable, safe to show
    details?: Record<string, unknown>;
  };
}
```

Each `ErrorCode` maps to exactly one HTTP status, as given in the tables above. The mapping lives in one exported `const` and is covered by a test. Adding a code requires amending this contract.

---

## 4. Data model

### 4.1 `messages` — stored message

Primary key: `(account_id, gmail_id)`. Upsert on conflict replaces every Gmail-sourced column and sets `updated_at` and `fetched_at`.

Source column paths refer to the Gmail `users.messages.get?format=full` response (`Message` resource). "Header X" means the first entry in `payload.headers` whose `name` matches X case-insensitively.

| Column | Postgres type | Nullable | Source in Gmail response | Notes |
|---|---|---|---|---|
| `account_id` | `uuid` | no | — (engine: JWT `sub`) | FK → `accounts.id`, `on delete cascade` |
| `gmail_id` | `text` | no | `id` | |
| `thread_id` | `text` | no | `threadId` | indexed |
| `label_ids` | `text[]` | no | `labelIds` | `{}` when absent |
| `is_unread` | `boolean` | no | derived: `labelIds` contains `"UNREAD"` | indexed with `account_id` |
| `snippet` | `text` | no | `snippet` | stored as returned (HTML entities not decoded); `''` when absent |
| `history_id` | `text` | no | `historyId` | uint64 kept as decimal string |
| `internal_date` | `timestamptz` | no | `internalDate` | epoch-ms string → timestamp; index `(account_id, internal_date desc)` |
| `size_estimate` | `integer` | no | `sizeEstimate` | |
| `subject` | `text` | yes | header `Subject` | RFC 2047 decoded |
| `from_address` | `jsonb` | yes | header `From` | `EmailAddress`, first mailbox |
| `to_addresses` | `jsonb` | no | header `To` | `EmailAddress[]`, `[]` when absent |
| `cc_addresses` | `jsonb` | no | header `Cc` | `EmailAddress[]` |
| `bcc_addresses` | `jsonb` | no | header `Bcc` | `EmailAddress[]`; usually present only on sent mail |
| `reply_to` | `jsonb` | no | header `Reply-To` | `EmailAddress[]` |
| `date_header` | `timestamptz` | yes | header `Date` | null if unparseable |
| `rfc822_message_id` | `text` | yes | header `Message-ID` | includes angle brackets |
| `in_reply_to` | `text` | yes | header `In-Reply-To` | |
| `references` | `text[]` | no | header `References` | split on whitespace; `{}` when absent |
| `body_text` | `text` | yes | first `payload` part (depth-first) with `mimeType = "text/plain"` and empty `filename` → `body.data` | base64url-decoded, then charset-decoded per that part's `Content-Type` |
| `body_html` | `text` | yes | first part with `mimeType = "text/html"` and empty `filename` → `body.data` | same decoding; stored unsanitized (sanitizing is the client's job) |
| `attachments` | `jsonb` | no | every part with non-empty `filename` and `body.attachmentId` → `{ partId, filename, mimeType, size: body.size, attachmentId: body.attachmentId }` | `AttachmentMeta[]`; bytes not fetched |
| `fetched_at` | `timestamptz` | no | — (engine: time of Gmail fetch) | |
| `created_at` | `timestamptz` | no | — (engine: `default now()`) | not changed on upsert |
| `updated_at` | `timestamptz` | no | — (engine: set on every upsert) | |

Not stored: `raw`, the full `payload.headers` array, attachment bytes.

DTO mapping (`Message` in §2): `gmail_id→id`, `thread_id→threadId`, `from_address→from`, `to_addresses→to`, `cc_addresses→cc`, `bcc_addresses→bcc`, `reply_to→replyTo`, `date_header→date`, the rest camelCased. `account_id`, `fetched_at`, `created_at` and `updated_at` are not exposed.

### 4.2 `accounts` — linked Gmail account

One Gmail account per engine user.

| Column | Postgres type | Nullable | Source | Notes |
|---|---|---|---|---|
| `id` | `uuid` | no | engine, `default gen_random_uuid()` | JWT `sub` |
| `google_sub` | `text` | no | `id_token.sub` | unique |
| `email` | `text` | no | `id_token.email` | lowercased |
| `refresh_token_enc` | `bytea` | no | token response `refresh_token` | AES-256-GCM: `iv(12) ‖ ciphertext ‖ tag(16)`, key = `TOKEN_ENCRYPTION_KEY` (32 bytes, base64) |
| `scopes` | `text[]` | no | token response `scope` | split on space |
| `created_at` | `timestamptz` | no | engine | |
| `updated_at` | `timestamptz` | no | engine | |

Access tokens are never persisted. They are cached in memory per account until 60 s before `expires_in` elapses.

Supabase notes: tables live in `public` with RLS **enabled and no policies**. The engine connects with the service-role key, so no client can reach them through PostgREST.

---

## 5. Repository interfaces (the S1 ↔ S2 seam)

Session 1 defines these in `src/store/types.ts` and ships the in-memory fakes plus a shared contract test suite (`src/store/repository.contract.ts`) that runs against any implementation.

```ts
interface AccountRepository {
  upsertByGoogleSub(input: {
    googleSub: string; email: string;
    refreshTokenEnc?: Uint8Array;   // omitted → keep existing
    scopes: string[];
  }): Promise<Account>;
  findById(id: string): Promise<Account | null>;
}

interface MessageRepository {
  upsertMany(accountId: string, messages: StoredMessage[]): Promise<void>;  // atomic
  findById(accountId: string, gmailId: string): Promise<StoredMessage | null>;
  updateLabels(accountId: string, gmailId: string, labelIds: string[]): Promise<StoredMessage | null>;
}
```

`StoredMessage` and `Account` are the camelCased row types from §4. All repository errors are thrown as `StoreError`, which routes map to `503 STORE_UNAVAILABLE`.

---

## 6. Sequencing rule — two-session build

**Rule: server logic ships and its tests pass before the schema is reviewed or merged.**

### Session 1 — Server
Owns: project scaffolding, Hono app, the four endpoints plus `GET /auth/google`, the Gmail client, MIME builder, body extraction, JWT, token encryption, the error envelope, the repository **interfaces**, in-memory fakes, the repository contract test suite, and all Vitest + msw tests.
Must not: create any SQL, Supabase migration, or Supabase client code.
Done when: AC-1 through AC-12 pass in CI, and the PR is merged to `main`.

### Session 2 — Schema
Owns: `supabase/migrations/<timestamp>_init.sql` implementing §4 exactly, `SupabaseAccountRepository` and `SupabaseMessageRepository`, and wiring them in behind the existing interfaces.
Must not: change any file under `src/routes`, `src/gmail`, or `src/store/types.ts`, or change the repository contract tests. If the interface is wrong, stop and amend this contract first.
Done when: AC-13 passes, meaning the contract test suite runs green against a local Supabase (`supabase start`), and all Session 1 tests are still green.

### Gates (enforced in order)
1. **G1** — Session 1 PR merged, with CI green on its merge commit.
2. **G2** — Session 2 may open its branch from `main` only after G1. Drafting earlier is allowed, but nothing is reviewed until G1 holds.
3. **G3** — The Session 2 PR is reviewed only after G1, and its description links the green Session 1 CI run.
4. **G4** — The Session 2 PR is merged only when its own CI (Session 1 suite + repository contract suite against Supabase) is green.

A schema review or merge that happens before G1 violates the contract (AC-14) and must be reverted.

---

## 7. Out of scope

Attachment download or upload, drafts, labels other than `UNREAD`, push notifications / `users.watch`, history-based sync, multiple Gmail accounts per user, JWT refresh endpoint, frontend, deployment.
