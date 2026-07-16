# BriefBay

Client intake portal: send a link, an LLM asks follow-up questions until the brief is actually buildable, you get a structured brief doc instead of a half-filled form.

**Status: planned — not yet built (50-SaaS challenge #69)**

## The problem

Agencies and freelancers lose days to back-and-forth emails clarifying vague client briefs ("make it pop", no budget, no timeline) before real work can start. A static form doesn't adapt when an answer is too thin.

## Target buyer

Small agencies and freelancers (design, dev, marketing) who run client intake themselves and want a brief they can act on, not a form response to interpret.

## Pricing hypothesis

Rs399/mo. One credit spent per finalized (buildable) brief, not per message — the conversation itself is free-flowing.

## Stack

Cloudflare Worker (TypeScript/ESM) serving static frontend + `/api/*` in one deploy. Supabase for magic-link auth (agency side), Postgres+RLS, credits ledger. A second, unauthenticated public route serves the client-facing chat via an unguessable per-intake token — not JWT-gated. LLM: Anthropic Claude, sync per conversation turn.

## How to continue this build

Nothing is implemented yet. Start with `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered TDD task list. `CLAUDE.md` points any coding agent at the shipped reference implementation to copy conventions from.

## Risks / constraints

- The client-facing intake is a **conversational form state machine**, not a static form — each turn calls the LLM to decide the next question or finalize; capped at a fixed max turns (e.g. 20) to bound cost, since the public route is unauthenticated.
- **The public per-client link is unauthenticated by design.** Its only protection is an unguessable token — this is a cost-DoS surface (someone could hammer the endpoint) that needs rate limiting before launch, called out explicitly in the LLD's security notes.
- Output is a **structured brief document** (JSON + rendered view), not free text — the finalize prompt must reliably hit the required-fields schema or the brief isn't usable.
- No client accounts, no email notifications, no e-signature in v1 — link-and-chat only.
