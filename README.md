# VibeMail Engine

A TypeScript backend that links a user's Gmail account through Google OAuth and exposes a small JSON API for listing, sending, and marking messages read or unread. Messages are always fetched live from Gmail and persisted to PostgreSQL on Supabase before each response.

> **Status:** pre-implementation. [`docs/CONTRACT.md`](docs/CONTRACT.md) is the source of truth for the build. Anything it doesn't specify is out of scope.

## Stack

- TypeScript (strict) on Node.js 24 LTS
- [Hono](https://hono.dev) HTTP framework
- PostgreSQL on Supabase, accessed only through repository interfaces
- Vitest + `msw` for tests, with no real network calls

## API

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/auth/google` | Redirect to Google's consent screen |
| `GET` | `/auth/google/callback` | Exchange the OAuth code and return an engine JWT |
| `GET` | `/messages` | List messages (`limit`, `pageToken`, `q`, `unread`) |
| `POST` | `/messages/send` | Send a message or a reply |
| `PATCH` | `/messages/:id` | Mark a message read or unread (`{ "read": boolean }`) |

Authenticated endpoints require `Authorization: Bearer <jwt>`. Every error response uses a single envelope, `{ "error": { "code", "message", "details?" } }`, with a closed set of codes. See contract §2–§3 for request and response types and the full error tables.

## Configuration

All configuration comes from environment variables. Copy `.env.example` to `.env` and fill in:

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` | Google OAuth client |
| `JWT_SECRET`, `JWT_TTL_SECONDS` | Signing key and lifetime for engine JWTs (TTL defaults to 86400) |
| `TOKEN_ENCRYPTION_KEY` | 32-byte base64 key for AES-256-GCM encryption of refresh tokens |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | Supabase project connection |

Never commit secrets.

## Scripts

```sh
npm run typecheck   # tsc --noEmit
npm run lint
npm test            # Vitest; Gmail mocked with msw
```

## Build plan

The build runs in two sequential sessions (contract §6):

1. **Server.** The app, endpoints, Gmail client, MIME and body handling, auth, repository interfaces, in-memory fakes, and all tests. No SQL.
2. **Schema.** The Supabase migration and the repository implementations, which must pass the same contract test suite as the fakes.

Schema work is not reviewed or merged until the server PR is merged with green CI.

## Out of scope

Attachment download or upload, drafts, labels other than `UNREAD`, push notifications, history-based sync, multiple Gmail accounts per user, JWT refresh, a frontend, and deployment.
