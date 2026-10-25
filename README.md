# VibeMail Engine

A TypeScript backend that links a user's Gmail account through Google OAuth and exposes a small JSON API for listing, sending, and marking messages read or unread. Message lists are always fetched live from Gmail and persisted to PostgreSQL on Supabase before each response. Gmail Pub/Sub push notifications keep the stored copy in sync between requests.

> **Status:** pre-implementation. [`docs/CONTRACT.md`](docs/CONTRACT.md) (v3) is the source of truth for the build. Anything it doesn't specify is out of scope.

## Stack

- TypeScript (strict) on Node.js 24 LTS
- Vercel Functions with file-based routing under `api/`, and no HTTP framework
- PostgreSQL on Supabase, accessed only through repository interfaces
- Gmail through a `MailProvider` interface (`src/providers/`)
- Jest + ts-jest + supertest for tests

## API

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/auth/google` | Redirect to Google's consent screen |
| `GET` | `/api/v1/auth/google/callback` | Exchange the OAuth code, start the Gmail watch, and return an engine JWT |
| `GET` | `/api/v1/messages` | List messages (`limit`, `pageToken`, `q`, `unread`) |
| `POST` | `/api/v1/messages/send` | Send a message or a reply |
| `PATCH` | `/api/v1/messages/:id` | Mark a message read or unread (`{ "read": boolean }`) |
| `POST` | `/webhook/gmail` | Pub/Sub push receiver; applies Gmail history deltas (a `vercel.ts` rewrite to `api/webhook/gmail.ts`) |
| `GET` | `/api/cron/renew-watch` | Daily Vercel cron that renews Gmail watches |

The `/api/v1` endpoints require `Authorization: Bearer <jwt>`, except the two auth routes (the callback is protected by its signed `state`). The webhook is authenticated by Pub/Sub's OIDC token, and the cron route by `CRON_SECRET`. Every error response uses a single envelope, `{ "error": { "code", "message", "details?" } }`, with a closed set of codes. See contract §2–§3 for request and response types and the full error tables.

## Configuration

All configuration comes from environment variables. Copy `.env.example` to `.env` and fill in:

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` | Google OAuth client. The redirect URI must match the deployed callback path |
| `GOOGLE_PUBSUB_TOPIC` | Pub/Sub topic that Gmail publishes mailbox changes to |
| `PUBSUB_PUSH_AUDIENCE`, `PUBSUB_PUSH_SERVICE_ACCOUNT` | Expected OIDC audience and service account on push requests |
| `JWT_SECRET`, `JWT_TTL_SECONDS` | Signing key and lifetime for engine JWTs (TTL defaults to 86400) |
| `TOKEN_ENCRYPTION_KEY` | 32-byte base64 key for AES-256-GCM encryption of refresh tokens |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | Supabase project connection |
| `CRON_SECRET` | Bearer secret Vercel sends to the cron route |
| `FRONTEND_URL` | Frontend origin. Reserved; no engine behavior depends on it yet |

## Scripts

Not wired yet; `package.json` still has the default `test` placeholder. The contract requires `typecheck`, `lint`, `test:unit` and `test:integration` (contract §1). Local preview runs with `vercel dev`.

## Build plan

The build follows seven units in order, listed in [`docs/build_sequence.md`](docs/build_sequence.md):

1. Provider abstraction interface
2. Gmail OAuth with a token persistence listener
3. Sync and read
4. Pub/Sub webhook receiver
5. Send
6. Vercel function entry points
7. Integration tests

The build runs in two sessions (contract §6):

- **Server logic** on `main` owns `src/` and `api/`, and builds the units in order, verifying each one before starting the next.
- **Schema** on the `schema` branch owns `supabase/migrations/` and `src/types/`.

The schema branch can't be merged until `npm test` exits 0 on `main`. Unit 7 (integration against live Supabase) starts after the schema merges.

## Out of scope

Attachment download or upload, drafts, labels other than `UNREAD` and `INBOX`, Pub/Sub pull subscriptions, an IMAP provider implementation (`src/providers/imap/` is reserved), multiple Gmail accounts per user, JWT refresh, a frontend, and production deployment.
