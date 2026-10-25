# VibeMail Engine — Atomic Build Sequence

Build each unit in order. Do not start the next unit until the current one is verified.

## 1. Provider abstraction interface
**Verified:** TypeScript compiles clean and the interface defines all required methods.

## 2. Gmail OAuth layer with token persistence listener
**Verified:** OAuth flow completes, tokens are stored in Supabase, and refresh works without a token mismatch.

## 3. Sync and read layer
**Verified:** Initial sync fetches 50 messages, and message objects match the CONTRACT.md data model exactly.

## 4. Pub/Sub webhook receiver
**Verified:** The webhook receives a notification and fetches the correct delta via history ID.

## 5. Send layer
**Verified:** A message sends successfully via the Gmail API on behalf of an authenticated user.

## 6. Vercel API function entry points
**Verified:** All four endpoints respond correctly in local preview.

## 7. Integration tests
**Verified:** The full test suite passes with no skipped tests against a live Supabase instance.
