# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

VibeMail Engine is a data liberation sync engine. It extracts Gmail data via OAuth2 into Supabase, exposes a REST API at `/api/v1`, and is deployed as Vercel serverless functions.

The project is **pre-implementation**: `src/`, `api/` and `tests/` contain only empty scaffold directories.

- `docs/CONTRACT.md` (v4) is binding. It holds endpoint shapes, error codes, the data model, interfaces, and acceptance criteria (AC-1…AC-19). Anything it doesn't specify is out of scope. If code needs a change to contract §2–§5, amend the contract in its own PR first.
- `docs/build_sequence.md` lists the seven build units in order, each with its verification check. Contract §6 maps each unit to its sections, ACs and migration.
- `README.md` summarizes v3. If it disagrees with the contract, the contract wins.

## Stack

- Node.js 24
- TypeScript, strict mode
- `googleapis` (Google APIs)
- `@supabase/supabase-js` (Supabase JS client)
- Jest with supertest
- Vercel (serverless functions, `vercel dev` for local preview)

## Commands

No npm scripts are wired yet (`npm test` is the default placeholder). The contract requires these, so add them when scaffolding:

- `npm run typecheck`: `tsc --noEmit`
- `npm run lint`
- `npm run test:unit`: Jest over `tests/unit`, with `googleapis` mocked via `jest.mock`, store faked, no network
- `npm run test:integration`: Jest over `tests/integration`, against a live Supabase instance
- Single test: `npx jest tests/unit/<file>.test.ts -t "<test name>"`
- `vercel dev`: local preview of the functions
- `supabase start`: local Supabase for integration tests

## Two-session architecture

| Session | Branch | Owns |
|---|---|---|
| Server logic | `main` | `src/` and `api/` |
| Schema | `schema` | the migrations folder (`supabase/migrations/`) and the types folder (`src/types/`) |

Each session writes only to the folders it owns.

## Sequencing rule

The schema session can't be merged until the server logic tests pass.

The schema session may draft at any time. Server units 1–6 run on in-memory fakes. Unit 7 (Supabase repositories and integration tests) starts only after the schema merges. The full gates are G1–G4 in contract §6.

## Never do

- Never use `any` as a TypeScript type.
- Never poll the Gmail API for new messages. Use Pub/Sub push webhooks. On-demand Gmail calls made to serve a request, such as `GET /api/v1/messages`, aren't polling.
- Never store OAuth tokens in plain text.
- Never make Gmail API calls without the Gmail API's OAuth2 client (`google.auth.OAuth2` from `googleapis`), which handles token refresh automatically.
- Never write to the `src/db` folder from the schema session.
- Never merge the schema session before `npm test` exits 0.
- Never hard-code credentials.

## Coding conventions

- All errors use the `docs/CONTRACT.md` error envelope shape: `{ error: { code, message, details? } }`. Codes form a closed union, each mapped to one HTTP status in a single exported const. Adding a code requires a contract amendment.
- All list endpoints use cursor-based pagination.
- All client endpoints use the `/api/v1` base path.
- All endpoints use JWT bearer token authentication, except the OAuth callback. The contract §2 Authentication table also exempts `/api/v1/auth/google` (login entry), `/webhook/gmail` (Pub/Sub OIDC token) and `/api/cron/renew-watch` (`CRON_SECRET`). Any new route requires the JWT unless the contract says otherwise.
- The Pub/Sub webhook endpoint lives at `/webhook/gmail`, not under `/api/v1`. It's served by `api/webhook/gmail.ts` through a `vercel.ts` rewrite. Never point clients, tests or the push subscription at `/api/webhook/gmail`.

## Architecture

There is no HTTP framework. Each route is a Vercel function file under `api/` that default-exports a `(req: VercelRequest, res: VercelResponse)` handler. The contract §2 routing table maps each path to its file. Handlers stay thin, and the logic lives in `src/`. Route tests wrap a handler in `http.createServer` for supertest.

The design has two seams; everything else depends only on the interfaces.

- **`MailProvider`** (contract §5.1, `src/providers/types.ts`) wraps all Gmail access, including OAuth, list/get, send, `setRead`, `watch` and `listHistory`.
  - `GmailProvider` (`src/providers/gmail/`) is the only implementation in scope. `src/providers/imap/` is reserved: leave it empty, and don't change the interface to suit it.
  - Raw Gmail errors never leave it; they're converted to a typed `ProviderError` whose code is from the §3 union.
  - It owns the **token persistence listener**: it subscribes to the OAuth2 client's `tokens` event and awaits the encrypted write of any rotated refresh token before releasing the new access token.
- **Repositories** (contract §5.2, `src/db/types.ts`): `AccountRepository` and `MessageRepository` have in-memory fakes and Supabase implementations. Both must pass one shared repository contract test suite. Sync writes go through `withSyncLock`, which is a transaction holding a per-account Postgres advisory lock.

Data flows:

- **Reads are live.** `GET /api/v1/messages` always calls Gmail, upserts the results, and only then responds; it returns a 503 rather than unsaved data. The synced store doesn't serve reads. This was a deliberate decision, so don't "optimize" list to read from the database.
- **Sync** keeps the store current in parallel:
  1. The OAuth callback calls `users.watch` *before* the initial 50-message sync, which runs via `waitUntil`, so no change slips between the two.
  2. The Pub/Sub webhook (verified by OIDC token) applies `history.list` deltas from `accounts.history_id`. Replays and stale `historyId`s are ignored.
  3. A cursor that's too old (`history.list` returns 404) triggers a full resync.
  4. A daily Vercel cron (`/api/cron/renew-watch`) renews watches without touching `history_id`.
- **Schema:** migrations are additive. Never edit a shipped migration. Tables have RLS enabled with no policies, and the engine uses the service-role key.

## Known issues

- **`tsconfig.json`:** `rootDir` is `./src`, but `include` also lists `api/**`, and ts-jest will compile `tests/`. `tsc` will error with TS6059 as soon as `api/` or `tests/` has TypeScript files. It's intentionally left as is for now, so expect to fix it when the first file lands outside `src/`.
- **Dependencies:** `@vercel/node` (handler types), `@vercel/functions` (`waitUntil`) and a linter aren't installed yet, though the contract relies on them.
