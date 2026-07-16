# BriefBay — Low-Level Design

## Architecture

```
Agency browser (static assets, magic-link auth via supabase-js CDN)
   |  JWT
   v
Cloudflare Worker (TypeScript, ESM, single deploy)
   |-- /api/*           JWT-verified agency routes (clients, intakes, dashboard)
   |-- /api/public/*    TOKEN-verified client routes (no Supabase auth at all)
   |-- Anthropic Messages API (sync, json_schema) per conversation turn + finalize
   v
Supabase Postgres (RLS on, default-deny for anon) via PostgREST (service-role key)

Client browser (public, no login) --token--> /api/public/intake/:token
```

Agency flow: `Dashboard -> POST /api/intakes (create client + intake, get shareable link)`.
Client flow: `open /intake/:token -> GET /api/public/intake/:token (current state) -> POST .../message per answer -> Worker calls LLM with transcript + required-fields checklist -> {done, next_question, extracted_fields} -> loop until done or max-turns cap -> finalize call produces structured_brief -> spend 1 credit from the agency's profile -> intake.status = 'ready'`.

Every turn is one Anthropic call, a few seconds — fully synchronous, no queue.

## Data model

```sql
create table profiles (
  id uuid primary key references auth.users(id),
  email text not null,
  credits int not null default 1,
  created_at timestamptz not null default now()
);

create table clients (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id),
  name text not null,
  email text,
  created_at timestamptz not null default now()
);

create table intakes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id),
  client_id uuid references clients(id),
  public_token uuid not null default gen_random_uuid() unique,
  title text not null,
  status text not null default 'draft'
    check (status in ('draft','in_progress','ready','completed')),
  structured_brief jsonb,
  turn_count int not null default 0,
  created_at timestamptz not null default now()
);

create table intake_messages (
  id uuid primary key default gen_random_uuid(),
  intake_id uuid not null references intakes(id) on delete cascade,
  role text not null check (role in ('assistant','user')),
  content text not null,
  created_at timestamptz not null default now()
);
```

State machine (`intakes.status`): `draft` (agency created it, link not yet opened) `-> in_progress` (first client message received) `-> ready` (LLM signals `done`, or `turn_count` hits the cap and the Worker force-finalizes) `-> completed` (agency marks it reviewed). Only `draft`/`in_progress` accept new client messages; `ready`/`completed` reject POSTs to the public route with 409.

RLS: `clients`/`intakes`/`intake_messages` scoped to `auth.uid() = user_id`, `select`/`update`/`delete` only — **and no policy grants `anon` any access to these tables at all**, default-deny. The public `/api/public/*` routes are not an RLS carve-out: the Worker uses the service-role key for every public-route read/write, and the token match (`public_token = :token`) is the Worker's own authorization check, done in application code before touching the row. `spend_credit`/`refund_credit` RPCs copied from the reference, revoked from `anon`/`authenticated`.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config` | GET | none | Public runtime config | — |
| `/api/clients` | POST/GET | JWT | Create/list agency's clients | — |
| `/api/intakes` | POST | JWT | Create intake for a client, generate `public_token`, return shareable link | — |
| `/api/intakes` | GET | JWT | List agency's intakes with status | — |
| `/api/intakes/:id` | GET/PATCH/DELETE | JWT | Detail (transcript+brief) / mark completed / delete | not owner -> 404 |
| `/api/public/intake/:token` | GET | token | Current state (title, transcript, status) for the client UI | invalid/completed token -> 404/409 |
| `/api/public/intake/:token/message` | POST | token | Submit client answer; LLM turn; may finalize | over turn cap -> force-finalize; oversized message -> 400 |

## LLM strategy

Provider: **Anthropic, claude-sonnet-4-6**, sync, strict JSON via `output_config` `json_schema`.

Turn prompt: system prompt carries the required-fields checklist (project type, goals, deliverables, budget range, timeline, target audience, stakeholders, inspiration references, constraints) plus the full transcript so far. Model returns whether enough is known to finalize, the single next question if not, and any fields it can already extract from the latest answer.

Turn output schema:
```json
{
  "type": "object",
  "properties": {
    "done": {"type": "boolean"},
    "next_question": {"type": ["string","null"]},
    "extracted_fields": {"type": "object"}
  },
  "required": ["done","next_question","extracted_fields"]
}
```

Finalize prompt: transcript + accumulated extracted fields -> one structured brief. Output schema: `{project_type, goals[], deliverables[], budget_range, timeline, target_audience, stakeholders[], inspiration_refs[], constraints[], raw_notes}`.

Cost estimate: ~₹0.5-1 per turn (transcript grows each turn, ~400-900 tokens); typical 5-8 turns to a finalized brief -> ~₹4-8 total LLM cost per completed intake. Only 1 credit is charged (at finalize) regardless of turn count, so the turn cap (see Security) is what bounds worst-case cost, not the credit system.

Sync vs async: **sync** for every call — each turn and the finalize call are single, fast Anthropic requests, nowhere near the Worker's ~100s cap.

## Frontend pages

Agency: Landing/pricing · Login (magic-link) · Dashboard (clients + intakes list with status) · New intake (create, copy shareable link) · Intake detail (transcript, structured brief, export/copy, mark completed) · Account (credits, manual UPI top-up).
Client (public, no login): `/intake/:token` chat-style single page — question, text input, submit; a "thanks, submitted" screen on finalize.

## Error handling + credits/refund

`spend_credit` happens only at finalize, wrapping the finalize Anthropic call; on failure, `refund_credit` and set `status` back to `in_progress` so the agency can retry rather than losing the transcript. Per-turn calls don't touch credits, so a turn failure just returns a retryable error to the client UI ("something went wrong, try answering again") with the transcript intact.

## Integrations and launch gates

None required for v1 — no external platform APIs, so nothing blocks launch on OAuth review or partnership approval. Email notification to the agency when a brief finalizes is a natural v1.1 (would need a transactional email provider) but is explicitly deferred to keep v1 minimal.

## Security notes

Default-deny RLS for `anon` on every table — the public route's only access path is the Worker's service-role key plus its own token check, which must run before any query, not after. `public_token` is a `uuid` (122 bits, unguessable by brute force). **Cost-DoS**: because `/api/public/intake/:token/message` requires no auth and triggers a real LLM call, it must be rate-limited (per-token, e.g. 1 message per 5 seconds) and hard-capped at `turn_count <= 20` (force-finalize past that) — this is the one route in the whole product where an anonymous request costs the operator money, and it needs its own guard, separate from the credits system. Message length capped at 2000 chars, HTML-stripped. Secrets via `wrangler secret put` only.

## Out of scope for v1

Client accounts/login; email notifications; e-signature or brief approval workflow; multiple intake templates per industry; editing the required-fields checklist per agency; exporting to PDF (copy/JSON export only); team/multi-seat agency accounts.
