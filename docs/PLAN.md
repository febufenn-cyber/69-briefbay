# BriefBay — Build Plan

TDD throughout: write the failing test named in each task before implementation. Each task is sized for one focused agent session. Work in order — later tasks depend on earlier interfaces.

**T1 — Supabase schema + RLS**
Files: `supabase/migrations/0001_init.sql`.
Interfaces: tables `profiles`, `clients`, `intakes`, `intake_messages` per LLD; `spend_credit`/`refund_credit` RPCs copied from the reference.
Test first: SQL script asserting `anon` has zero privileges (no select/insert/update/delete) on all four tables, and a foreign-user `select` via `authenticated` role returns zero rows for `intakes`.
Done when: migration applies cleanly; both RLS tests pass.

**T2 — Worker skeleton + agency auth middleware**
Files: `src/index.ts`, `src/auth.ts`, `wrangler.toml`.
Interfaces: `verifyJwt(req: Request, env: Env): Promise<{ sub: string; email: string } | null>`; `GET /config`.
Test first: vitest mocking `fetch` to GoTrue — valid/invalid token cases.
Done when: `wrangler dev` serves `/config`; auth test passes.

**T3 — Agency routes: clients + intake creation**
Files: `src/routes/clients.ts`, `src/routes/intakes.ts`.
Interfaces: `createIntake(userId, clientId, title): Promise<{ id, publicToken }>`; `listIntakes(userId): Promise<IntakeSummary[]>`.
Test first: creating an intake returns a `public_token` that is a valid uuid and not guessable from the intake `id`.
Done when: create/list/detail/patch/delete tests pass, all scoped to the JWT's `sub`.

**T4 — Public token gate + turn logic**
Files: `src/routes/public-intake.ts`, `src/llm/turn.ts`.
Interfaces: `resolveByToken(token: string): Promise<Intake | null>` (service-role fetch, no RLS involved); `runTurn(intake, transcript, apiKey): Promise<{ done, next_question, extracted_fields }>`.
Test first: an invalid token returns 404 without touching any real row (no timing side-channel test needed at this stage, but assert the query is a single indexed lookup); a `completed`-status intake rejects a POST message with 409.
Done when: token-gate tests pass; mocked-LLM turn test (valid/malformed/HTTP-error Anthropic responses) passes.

**T5 — Turn cap + rate limit + finalize**
Files: `src/routes/public-intake.ts` (extended), `src/llm/finalize.ts`, `src/ratelimit.ts`.
Interfaces: `enforceTurnCap(intake): boolean` (true once `turn_count >= 20`, forces finalize); `checkRateLimit(token: string): Promise<boolean>`; `finalizeBrief(transcript, fields, apiKey): Promise<StructuredBrief>`.
Test first: the 21st message on an intake force-finalizes instead of asking another question, regardless of the LLM's `done` value; two messages within the rate-limit window on the same token — the second is rejected before any LLM call is made (assert on call count, not just response).
Done when: both tests pass; finalize spends exactly 1 credit and refunds on failure, leaving `status` at `in_progress` for retry.

**T6 — Frontend: agency dashboard**
Files: `public/index.html`, `public/app.js`.
Interfaces: none new (calls T2/T3 routes).
Test first: manual — magic-link round trip against a Supabase test project.
Done when: a logged-in agency can create a client, create an intake, and copy the shareable `/intake/:token` link.

**T7 — Frontend: client chat**
Files: `public/intake.html`, `public/intake.js`.
Interfaces: none new (calls T4/T5 public routes).
Test first: manual — opening a fresh token shows the first question; submitting answers advances the conversation; hitting finalize shows the "thanks" screen.
Done when: full conversation completes end-to-end against a local Worker with a real (test) Anthropic key.

**T8 — Frontend: intake detail + billing**
Files: `public/intake-detail.html`, `public/account.html`.
Done when: agency can view the finalized structured brief and export/copy it; credit balance and manual UPI top-up instructions match the reference's day-1 pattern.

**T9 — Deploy + live smoke test + launch checklist**
Files: `scripts/smoke.ts`, `wrangler.toml` (production vars).
Interfaces: `smoke.ts` runs signup-stub -> create intake -> simulate client conversation to finalize -> verify brief against the deployed URL, exits non-zero on failure.
Done when: `wrangler deploy` succeeds; smoke script is green in production; launch checklist confirmed:
- Pricing page live at Rs399/mo
- Rate limit and turn cap verified live against the public route (not just mocked)
- All secrets set via `wrangler secret put` (none committed in `.dev.vars`)
- RLS spot-checked in production: `anon` key confirmed to get zero rows on all four tables

## Notes

T4/T5 are the highest-risk tasks — the public token gate is the product's only real attack surface, since every other route sits behind Supabase auth. Do not merge T5 without the rate-limit test actually asserting the LLM was not called on a blocked request, not just that the HTTP response was a 429; a test that only checks the response code can pass while still leaking a paid API call per rejected request.
