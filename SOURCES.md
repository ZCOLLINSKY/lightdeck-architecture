# Sources

Paths are in the private `lighting-os` repository at the production release of 2026-09-02. Listed so every claim in the README is traceable.

| Claim | Path |
|---|---|
| 46 serverless routes | `api/*.js` (count of files) |
| Browser has no direct data path; Google Sign-In + magic links + sessions in Postgres; all data via API with service role | `api/auth.js` header, `scripts/migrations/2026-07-16-public-schema-lockdown.sql` header, `lib/api-shared.js` |
| RLS enabled and forced, browser roles revoked | `scripts/migrations/2026-06-19-rls-lockdown.sql`, `scripts/migrations/2026-07-16-public-schema-lockdown.sql` |
| Event trigger locks down future public objects | `scripts/migrations/2026-07-16-public-schema-lockdown.sql` (lines ~119-140, ~456-470) |
| Durable rate limit RPC, fail closed on Vercel | `scripts/migrations/2026-07-16-durable-rate-limit-ledger.sql`, `lib/rate-limit.js`, `lib/usage.js` |
| Render attempt CAS store, 7-day retention, 42702 fix | `scripts/migrations/2026-08-16-render-attempt-idempotency.sql`, `scripts/migrations/2026-08-17-fix-claim-render-attempt-ambiguity.sql`, `lib/render-attempt-store.js` |
| Stripe event claim with lease | `scripts/migrations/2026-07-16-stripe-event-claims.sql` |
| Paid-effects outbox in the same PATCH; per-channel lease | `scripts/migrations/2026-07-15-invoice-effect-outbox.sql`, `scripts/migrations/2026-07-17-invoice-effect-lease.sql`, `lib/invoice-paid-effects.js` |
| Proposal approval outbox | `scripts/migrations/2026-07-16-proposal-approval-outbox.sql` |
| Gap-free invoice numbers | `scripts/migrations/2026-06-17-invoice-seq.sql` |
| Idempotent proposal creation | `scripts/migrations/2026-07-16-proposal-create-idempotency.sql`, `scripts/migrations/2026-09-02-proposals-idempotency-index.sql` |
| Append-only render receipts with mutation trigger | `scripts/migrations/2026-08-01-render-receipts.sql`, `lib/render-receipt-store.js` |
| Onboarding state machine | `supabase/migrations/20260830193045_lightdeck_onboarding_state_machine.sql`, `lib/onboarding-state.js` |
| Immutable source job id, legacy quarantine | `scripts/migrations/2026-08-21-proposal-client-as-built-atomic.sql`, `lib/as-built.js` |
| Tenant tuple in rendering spine; cross-tenant cache hit | `scripts/migrations/2026-07-30-render-source-spine.sql` header; `lib/render-source-spine.js`, `lib/render-source-manifest.js` |
| Single-tenant flag | `LIGHTDECK_SINGLE_TENANT` in `api/` and `lib/tenant-identity.js` |
| AI spend caps and verifier circuit | `LIGHTDECK_ACCT_DAILY_UNITS`, `LIGHTDECK_AI_ORG_MONTHLY_USD`, `LIGHTDECK_RENDER_VERIFIER_CIRCUIT_MS` and related env in `api/render.js`, `api/verify-render.js`, `lib/render-finisher-budget.js` |
| Outbound gate | `scripts/migrations/2026-09-02-outbound-gate.sql`, `lib/outbound-gate.js` |
| Render budgets with retry reserve | `LD_RENDER_BUDGET_MS`, `LD_RENDER_RETRY_RESERVE_MS` in `api/render.js` |
| AI ownership boundary, JobTruthPacket | `docs/LIGHTDECK_AI_RUNTIME_ARCHITECTURE.md` |
| Test lane counts | `tests/regressions`, `tests/e2e`, `tests/visual`, `tests/mobile`, `tests/a11y`, `tests/adversarial` |
| Gate jobs and self-hosted runners | `.github/workflows/gate.yml` (`runs-on: [self-hosted, lightdeck-ci]`, foundation + 4 browser shards) |
| Compact receipts after storage exhaustion | `.github/workflows/gate.yml` ("Build compact foundation evidence") |
| Daily logical backup, heartbeat | `api/backup-supabase.js`, `lib/supabase-logical-backup.js`, `docs/supabase-backup-restore.md` |
| Incident rollback runbook | `docs/incident-rollback.md` |
| Production migration go/no-go | `docs/supabase-production-migration-2026-07-16.md`, `scripts/verify-production-migrations.mjs` |
| Hardening audit waves | PRs #227 and #228 on the private repo |
| Commit and day counts | `git rev-list --count HEAD` = 2785; first commit 2026-05-27, release 2026-09-02 |
