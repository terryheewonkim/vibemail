# VibeMail Engine — Build Contract

Status: **Draft v2** · Owner: Terry Kim · Date: 2026-10-25

This document is the single source of truth for the VibeMail engine build. Anything not specified here is out of scope. Changes require an amendment to this file, merged before the code that depends on them. The ordered build units and their verification checks live in [build_sequence.md](docs/build_sequence.md). §6 binds them to this contract.

**v2 amendments:** Pub/Sub push sync added (§2.5, §2.6, §4.2). Hosting moved to Vercel Functions (§0). Integration tests run against live Supabase (§1). Provider abstraction added (§5.1). Schema now ships with the unit that first needs it, replacing the server-first two-session rule (§6).

---

## 0. Fixed decisions

| Area | Decision |
|---|---|
| Language / runtime | TypeScript (strict), Node.js 24 LTS |
| HTTP framework | Hono app, exported as the handler for Vercel Functions |
| Hosting | Vercel Functions, Node.js runtime with Fluid Compute. Local preview: `vercel dev` |
| Store | PostgreSQL on Supabase, accessed only through repository interfaces (§5.2) |
| Mail provider | Accessed only through the `MailProvider` interface (§5.1). Gmail is the sole implementation |
| Client auth | Engine-issued JWT, sent as `Authorization: Bearer <token>` |
| Google OAuth scopes | `openid`, `email`, `https://www.googleapis.com/auth/gmail.modify` |
| Gmail read depth | `users.messages.get` with `format=full`; bodies decoded, attachment **metadata** only |
| Sync model | On link, an initial sync stores the 50 most recent messages. After that, Gmail Pub/Sub push notifications drive incremental sync through `users.history.list` |
| List source of truth | Gmail, live on every call; results upserted into the store before responding (unchanged from v1) |
| Error format | Custom envelope with a closed code union (§3) |
| Tests | Vitest. Unit tests: Gmail mocked with `msw` and the store mocked with in-memory fakes. Integration tests: live Supabase, with Gmail still mocked |
| Build | Seven units in order, per build_sequence.md (§6) |

---

## 1. Acceptance criteria

The project is **complete** when every item below is true. Each is verifiable by a command or a named test.

### Build and quality
- **AC-1** `npm run typecheck` (`tsc --noEmit`, `strict: true`) exits 0.
- **AC-2** `npm run lint` exits 0.
- **AC-3** `npm run test:unit` exits 0 with no skipped or `.only` tests and no real outbound network calls (`msw` configured with `onUnhandledRequest: "error"`).
- **AC-4** No secret is committed. All configuration is read from env vars listed in `.env.example`: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI`, `JWT_SECRET`, `JWT_TTL_SECONDS`, `TOKEN_ENCRYPTION_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `GMAIL_PUBSUB_TOPIC`, `PUBSUB_PUSH_AUDIENCE`, `PUBSUB_PUSH_SERVICE_ACCOUNT`, `CRON_SECRET`.

### Endpoint behavior (each has at least one passing route-level test)
- **AC-5** OAuth callback exchanges a valid `code`, creates or updates the account, stores the refresh token encrypted, starts a Gmail watch, and returns a JWT that authenticates the core endpoints.
- **AC-6** Every error case listed in §2 for every endpoint has a test that asserts **both** the HTTP status **and** the `error.code`.
- **AC-7** Message list returns messages in Gmail order and supports `limit`, `pageToken`, `q`, and `unread`. Afterward, every returned message is in the store with fields matching §4.
- **AC-8** Message send produces a valid RFC 5322 message (asserted by decoding the `raw` sent to the Gmail mock): correct `To`/`Cc`/`Bcc`/`Subject`, `multipart/alternative` when `html` is provided, and `In-Reply-To`/`References`/`threadId` when `replyToMessageId` is provided. The sent message is upserted and returned.
- **AC-9** Mark-as-read/unread is idempotent: calling it twice with the same value returns 200 both times and leaves Gmail labels and the stored `is_unread` consistent.
- **AC-10** No response body ever contains a Google access token, a refresh token, or material derived from `TOKEN_ENCRYPTION_KEY`.

### Tokens and sync
- **AC-11** An expired Google access token is refreshed transparently. When Google rotates the refresh token, the token listener persists the new one before any further Gmail call, and the next refresh uses it. A test asserts that the stored and in-use refresh tokens never diverge. A revoked refresh token (`invalid_grant`) yields `401 GMAIL_REAUTH_REQUIRED`.
- **AC-12** Initial sync stores exactly `min(50, mailbox size)` messages, newest first, each matching §4 field for field. `accounts.history_id` is set from the `watch` response, and `sync_status` becomes `ready`.
- **AC-13** Given a push notification, the webhook calls `history.list` from the stored `history_id` and applies exactly the delta: added messages are upserted, deleted messages are removed, and label changes update `label_ids`/`is_unread`. It then advances `history_id`. Replaying the same notification changes nothing.
- **AC-14** A stale cursor (`history.list` 404) triggers a full resync of 50 messages and resets `history_id`.

### Data
- **AC-15** The body-extraction function is unit-tested against fixtures for: plain-only, HTML-only, `multipart/alternative`, `multipart/mixed` with attachment, nested multipart, and a non-UTF-8 charset.
- **AC-16** Supabase migrations create exactly the tables and columns in §4. The Supabase repositories pass the **same** repository contract suite that the in-memory fakes pass.

### Deployment and integration
- **AC-17** Under `vercel dev`, the four core endpoints, plus `GET /auth/google`, `POST /webhooks/gmail` and `GET /cron/renew-watch`, respond with the statuses and shapes in §2.
- **AC-18** `npm run test:integration` exits 0 with no skipped tests against a live Supabase instance (local `supabase start` or a dedicated test project, never production), with migrations applied fresh. Gmail stays mocked.

### Process
- **AC-19** Every unit in build_sequence.md was verified before the next unit started (§6).

---

## 2. API endpoint contracts

All endpoints return `application/json`. Core endpoints that need auth require `Authorization: Bearer <jwt>`. Field types are TypeScript.

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

// Response DTO for a message. Mirrors §4.1 minus internal columns.
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

### Errors common to all JWT-authenticated endpoints

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

> **Supporting route:** `GET /auth/google` 302-redirects to Google's consent URL with `access_type=offline`, `prompt=consent`, the scopes in §0, and a `state` value. `state` is a stateless HMAC-signed token (`JWT_SECRET`) containing a random nonce and a 10-minute expiry.

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
6. Call `users.watch` with `{ topicName: GMAIL_PUBSUB_TOPIC, labelIds: ["INBOX"], labelFilterBehavior: "include" }`. Store `history_id` and `watch_expiration` from the response, and set `sync_status = "pending"`. A watch failure fails the request with `502 GMAIL_WATCH_FAILED`.
7. Schedule the initial sync (§2.7) with `waitUntil` from `@vercel/functions`, and respond without waiting for it.
8. Issue JWT: HS256, `sub = account.id`, `iat`, `exp = iat + JWT_TTL_SECONDS` (default 86400).

**Response** `200 OK`

```ts
interface OAuthCallbackResponse {
  token: string;                    // engine JWT
  tokenType: "Bearer";
  expiresAt: ISODateTime;
  account: { id: string; email: string; syncStatus: "pending" | "ready" | "failed" };
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
| 502 | `GMAIL_WATCH_FAILED` | `users.watch` returns an error (usually a misconfigured topic or IAM) |
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

### 2.5 Pub/Sub webhook receiver (supporting)

`POST /webhooks/gmail`

Called by a Cloud Pub/Sub **push** subscription on `GMAIL_PUBSUB_TOPIC`. This endpoint doesn't use the engine JWT.

**Auth:** `Authorization: Bearer <Google-signed OIDC token>`. The engine verifies the signature against Google's certs, checks that `aud` equals `PUBSUB_PUSH_AUDIENCE`, and requires `email_verified` with `email` equal to `PUBSUB_PUSH_SERVICE_ACCOUNT`.

**Request** (JSON body, a Pub/Sub push envelope)

```ts
interface PubSubPushRequest {
  message: {
    data: string;                   // base64url JSON: { emailAddress: string; historyId: string }
    messageId: string;              // Pub/Sub id, unrelated to Gmail ids
    publishTime: ISODateTime;
  };
  subscription: string;
}
```

**Behavior**
1. Decode `message.data` and look up the account by `emailAddress`. Unknown address: ack with `204` and do nothing.
2. If the notification's `historyId` ≤ the stored `history_id`, ack with `204` (duplicate or stale).
3. Take a per-account lock (`pg_advisory_xact_lock` on the account id) so concurrent notifications don't double-apply.
4. Call `users.history.list` with `startHistoryId = stored history_id` and `historyTypes = messageAdded, messageDeleted, labelAdded, labelRemoved`, following `nextPageToken` to the end.
5. Apply the delta. `messagesAdded`: fetch `format=full` and upsert. `messagesDeleted`: delete the row. `labelsAdded`/`labelsRemoved`: set `label_ids` and `is_unread` from the record's `message.labelIds`.
6. Set `history_id` to the `historyId` returned by `history.list`, in the same transaction as the delta.
7. If `history.list` returns 404 (cursor too old), run the full resync (§2.7) and reset `history_id`.

**Response** `204 No Content` on success or when ignored. Any 2xx acks the message. Non-2xx responses make Pub/Sub retry with backoff.

**Errors**

| HTTP | `error.code` | When |
|---|---|---|
| 401 | `PUSH_AUTH_INVALID` | OIDC token missing, bad signature, wrong `aud`, or wrong service account |
| 400 | `VALIDATION_FAILED` | Envelope malformed or `data` not decodable to the expected JSON. Pub/Sub retries every non-2xx response, so the subscription must have a dead-letter topic (max 5 delivery attempts) to stop endless redelivery |
| 401 | `GMAIL_REAUTH_REQUIRED` | Account's refresh token revoked; `sync_status` set to `failed` |
| 429 / 502 / 503 | `GMAIL_RATE_LIMITED` / `GMAIL_UNAVAILABLE` / `STORE_UNAVAILABLE` | As in the common table; Pub/Sub retries |

---

### 2.6 Watch renewal (supporting)

`GET /cron/renew-watch`

A Gmail watch expires after 7 days. A Vercel cron job (`vercel.ts` `crons`, daily at `0 3 * * *`) calls this route.

**Auth:** `Authorization: Bearer <CRON_SECRET>`.

**Behavior:** for every account whose `watch_expiration` is less than 48 h away, re-call `users.watch` and update `watch_expiration`. `history_id` is **not** overwritten, so no delta is lost. Each account is handled independently, and one failure doesn't stop the rest.

**Response** `200 OK`

```ts
interface RenewWatchResponse {
  renewed: number;
  failed: { accountId: string; code: ErrorCode }[];
}
```

**Errors:** `401 CRON_AUTH_INVALID` for a missing or incorrect secret, and `503 STORE_UNAVAILABLE` if accounts can't be listed.

---

### 2.7 Initial sync and full resync (internal, no route)

`syncInitial(accountId)` runs from the OAuth callback via `waitUntil`, and on a stale cursor (§2.5 step 7).
1. `users.messages.list` with `maxResults=50` and `includeSpamTrash=false`.
2. `users.messages.get?format=full` for each result (concurrency ≤ 10), mapped per §4.1 and upserted in one call.
3. On success, set `sync_status = "ready"` and `last_synced_at = now()`. On failure, set `sync_status = "failed"`. A failed sync is retried on the next push notification, which runs a full resync when `sync_status = "failed"`.

The watch is always started **before** the initial list, so changes that land between the two calls are captured by history starting from the watch's `historyId`.

---

## 3. Error envelope

Every non-2xx response, except a `204`, has exactly this shape:

```ts
type ErrorCode =
  | "VALIDATION_FAILED"
  | "AUTH_MISSING" | "AUTH_INVALID" | "AUTH_EXPIRED"
  | "OAUTH_STATE_INVALID" | "OAUTH_ACCESS_DENIED" | "OAUTH_CODE_INVALID"
  | "OAUTH_NO_REFRESH_TOKEN" | "OAUTH_EXCHANGE_FAILED"
  | "GMAIL_REAUTH_REQUIRED" | "GMAIL_SCOPE_INSUFFICIENT" | "GMAIL_RATE_LIMITED"
  | "GMAIL_UNAVAILABLE" | "GMAIL_SEND_REJECTED" | "GMAIL_WATCH_FAILED"
  | "INVALID_PAGE_TOKEN" | "MESSAGE_NOT_FOUND" | "REPLY_TARGET_NOT_FOUND"
  | "PAYLOAD_TOO_LARGE"
  | "PUSH_AUTH_INVALID" | "CRON_AUTH_INVALID"
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
| `account_id` | `uuid` | no | — (engine: JWT `sub`, or the account resolved from the push `emailAddress`) | FK → `accounts.id`, `on delete cascade` |
| `gmail_id` | `text` | no | `id` | |
| `thread_id` | `text` | no | `threadId` | indexed |
| `label_ids` | `text[]` | no | `labelIds` (or the history record's `message.labelIds` on label changes) | `{}` when absent |
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

### 4.2 `accounts` — linked Gmail account and sync cursor

One Gmail account per engine user.

| Column | Postgres type | Nullable | Source | Notes |
|---|---|---|---|---|
| `id` | `uuid` | no | engine, `default gen_random_uuid()` | JWT `sub` |
| `google_sub` | `text` | no | `id_token.sub` | unique |
| `email` | `text` | no | `id_token.email` | lowercased; unique; used to resolve push notifications |
| `refresh_token_enc` | `bytea` | no | token response `refresh_token`, or a rotated token from the `tokens` listener | AES-256-GCM: `iv(12) ‖ ciphertext ‖ tag(16)`, key = `TOKEN_ENCRYPTION_KEY` (32 bytes, base64) |
| `scopes` | `text[]` | no | token response `scope` | split on space |
| `history_id` | `text` | yes | `users.watch` response `historyId`, then `users.history.list` response `historyId` | sync cursor; uint64 as decimal string; null before first watch |
| `watch_expiration` | `timestamptz` | yes | `users.watch` response `expiration` | epoch-ms string → timestamp |
| `sync_status` | `text` | no | engine | check in (`'pending'`, `'ready'`, `'failed'`); default `'pending'` |
| `last_synced_at` | `timestamptz` | yes | engine | set after each successful initial sync or delta |
| `created_at` | `timestamptz` | no | engine | |
| `updated_at` | `timestamptz` | no | engine | |

Access tokens are never persisted. They are cached in memory per account until 60 s before `expires_in` elapses. On Fluid Compute that cache is per instance; a cache miss just triggers a refresh.

Supabase notes: tables live in `public` with RLS **enabled and no policies**. The engine connects with the service-role key, so no client can reach them through PostgREST.

---

## 5. Interfaces

### 5.1 Provider abstraction (unit 1)

All mail operations go through this interface, defined in `src/provider/types.ts`. `GmailProvider` is the only implementation. Routes and sync code import only the interface.

```ts
interface ProviderTokens {
  accessToken: string;
  accessTokenExpiresAt: Date;
  refreshToken?: string;            // present on initial exchange or rotation
  scopes: string[];
  idToken?: { sub: string; email: string };
}

interface ListPage { ids: string[]; nextPageToken: string | null; resultSizeEstimate: number }

interface HistoryDelta {
  added: string[];                                  // message ids to fetch and upsert
  deleted: string[];                                // message ids to delete
  labelChanges: { id: string; labelIds: string[] }[];
  historyId: string;                                // new cursor
}

interface MailProvider {
  // auth
  buildAuthUrl(state: string): string;
  exchangeCode(code: string): Promise<ProviderTokens>;
  onTokens(listener: (accountId: string, tokens: ProviderTokens) => Promise<void>): void;  // token persistence listener

  // read
  listMessages(accountId: string, q: { limit: number; pageToken?: string; query?: string; unreadOnly?: boolean }): Promise<ListPage>;
  getMessage(accountId: string, id: string): Promise<StoredMessage | null>;  // null on 404; already mapped per §4.1

  // write
  sendMessage(accountId: string, mime: Uint8Array, threadId?: string): Promise<{ id: string; threadId: string }>;
  setRead(accountId: string, id: string, read: boolean): Promise<{ labelIds: string[] }>;

  // sync
  watch(accountId: string): Promise<{ historyId: string; expiration: Date }>;
  listHistory(accountId: string, startHistoryId: string): Promise<HistoryDelta | "CURSOR_EXPIRED">;
}
```

Every method throws a typed `ProviderError` whose `code` is one of the `GMAIL_*`, `OAUTH_*`, `MESSAGE_NOT_FOUND` or `INVALID_PAGE_TOKEN` values from §3. Raw Gmail errors never escape the provider.

**Token persistence listener:** `GmailProvider` holds one `OAuth2Client` per account and subscribes to its `tokens` event. When an event carries a `refresh_token`, the registered `onTokens` listener encrypts it and writes it to `accounts.refresh_token_enc`. The provider awaits that write before releasing the new access token, so an in-flight call can never use a refresh token the store doesn't have.

### 5.2 Repositories

Defined in `src/store/types.ts`, with in-memory fakes and a shared contract suite (`src/store/repository.contract.ts`) that runs against both the fakes and Supabase.

```ts
interface AccountRepository {
  upsertByGoogleSub(input: {
    googleSub: string; email: string;
    refreshTokenEnc?: Uint8Array;   // omitted → keep existing
    scopes: string[];
  }): Promise<Account>;
  findById(id: string): Promise<Account | null>;
  findByEmail(email: string): Promise<Account | null>;
  setRefreshToken(id: string, refreshTokenEnc: Uint8Array): Promise<void>;
  setWatch(id: string, w: { historyId?: string; expiration: Date }): Promise<void>;
  setSyncStatus(id: string, status: "pending" | "ready" | "failed"): Promise<void>;
  listWatchesExpiringBefore(t: Date): Promise<Account[]>;
  // Runs fn inside one transaction holding the per-account advisory lock.
  withSyncLock<T>(id: string, fn: (tx: { messages: MessageRepository; accounts: AccountRepository }) => Promise<T>): Promise<T>;
}

interface MessageRepository {
  upsertMany(accountId: string, messages: StoredMessage[]): Promise<void>;  // atomic
  findById(accountId: string, gmailId: string): Promise<StoredMessage | null>;
  updateLabels(accountId: string, gmailId: string, labelIds: string[]): Promise<StoredMessage | null>;
  deleteMany(accountId: string, gmailIds: string[]): Promise<void>;
  setHistoryCursor(accountId: string, historyId: string): Promise<void>;  // inside withSyncLock only
}
```

`StoredMessage` and `Account` are the camelCased row types from §4. All repository errors are thrown as `StoreError`, which routes map to `503 STORE_UNAVAILABLE`.

---

## 6. Build sequence and gates

The build follows the seven units in [build_sequence.md](docs/build_sequence.md), strictly in order. A unit is **verified** when its check passes **and** the acceptance criteria tied to it below pass. The next unit doesn't start until then.

Schema ships with the unit that first needs it, not after the server. *This replaces v1's rule that server logic ships and passes tests before the schema is reviewed or merged.*

| # | Unit | Verification (from build_sequence.md) | Contract scope | ACs | Migration shipped |
|---|---|---|---|---|---|
| 1 | Provider abstraction interface | TypeScript compiles clean and the interface defines all required methods | §5.1 `MailProvider`, §5.2 interfaces + fakes, §3 error union | AC-1, AC-2 | — |
| 2 | Gmail OAuth layer with token persistence listener | OAuth flow completes, tokens are stored in Supabase, and refresh works without a token mismatch | §2.1 steps 1–5 and 8, `GET /auth/google`, token listener (§5.1) | AC-5 (minus watch), AC-10, AC-11 | `0001_accounts.sql`: §4.2 columns `id` … `scopes`, `created_at`, `updated_at` |
| 3 | Sync and read layer | Initial sync fetches 50 messages, and message objects match the contract data model exactly | §2.1 step 7, §2.7, §2.2, body extraction | AC-7, AC-12, AC-15 | `0002_messages.sql`: all of §4.1, plus §4.2 `sync_status`, `last_synced_at` |
| 4 | Pub/Sub webhook receiver | The webhook receives a notification and fetches the correct delta via history ID | §2.1 step 6, §2.5, §2.6 | AC-13, AC-14 | `0003_sync_cursor.sql`: §4.2 `history_id`, `watch_expiration`; unique index on `email` |
| 5 | Send layer | A message sends successfully via the Gmail API on behalf of an authenticated user | §2.3, §2.4 | AC-8, AC-9 | — |
| 6 | Vercel API function entry points | All four endpoints respond correctly in local preview | Hono app exported for Vercel, `vercel.ts` (cron), all routes in §2 | AC-17 | — |
| 7 | Integration tests | The full test suite passes with no skipped tests against a live Supabase instance | Full suite, repository contract suite against Supabase | AC-3, AC-6, AC-16, AC-18 | — |

### Gate rules
1. **Live checks.** Units 2, 4 and 5 are verified once against a real Google test account and the dev Supabase project, plus automated tests with mocked Gmail. The live run is recorded (date, account, result) in the unit's PR description.
2. **Migrations are additive.** Each migration is a new file. No unit edits a migration that an earlier unit shipped. A schema change after merge gets a new numbered migration and a contract amendment.
3. **One unit per PR.** Each PR's description states the unit number, the verification result, and the ACs it satisfies. A PR for unit *n* isn't reviewed until unit *n − 1* is merged.
4. **Contract first.** If a unit needs a change to §2–§5, amend this file in its own PR, merged before the unit's PR.

---

## 7. Out of scope

Attachment download or upload, drafts, labels other than `UNREAD` and `INBOX` (watch filter), Pub/Sub pull subscriptions, multiple Gmail accounts per user, JWT refresh endpoint, frontend, and production deployment and promotion (only local `vercel dev` preview is required).
