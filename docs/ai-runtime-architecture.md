# LightDeck AI Runtime Architecture

Status: architecture decision record and implementation contract
Date: 2026-08-01
Scope: LightDeck-owned proposals, invoices, PDFs, photo analysis, fixture plans, and night renders

## Executive decision

LightDeck is the system of record and artifact compiler. AI providers are replaceable workers.

The final proposal, invoice, PDF, share page, fixture plan, render receipt, pricing, deposit, warranty, and client identity must be generated and validated by LightDeck from canonical project data. A model may propose bounded copy, classifications, placement suggestions, or image pixels. It may never become the source of truth for money, identity, legal terms, project state, or what gets sent.

This is the product promise:

> LightDeck creates the deliverable. The selected AI engine helps with specific bounded steps, and LightDeck proves what facts and design produced the result.

The production runtime should support three routes behind one contract:

1. **LightDeck Managed AI**: server-side API usage, metered to a real server ledger. This is the default for reliable web and mobile use.
2. **Contractor BYOK API**: the contractor pays OpenAI or Anthropic directly. Credentials live in an OS keychain or a purpose-built encrypted secret vault, never browser storage.
3. **Use my AI membership**: a LightDeck plugin/MCP integration used from Codex or Claude Code. A later local companion may support user-initiated jobs from the LightDeck UI only after provider terms are confirmed. Provider credentials never pass through LightDeck.

Consumer memberships are not interchangeable with API credit. Codex can use ChatGPT subscription authentication locally, and Claude Code can use Claude Pro/Max authentication locally, but neither subscription becomes a generic quota that LightDeck's hosted API can consume. Production background automation and commercial embedding should use provider APIs unless the provider explicitly approves another route.

## Why the current implementation cannot be the foundation

The repository already has useful safety work: authenticated shared-key routes, durable usage guards, bring-your-own-key provider checks, render design hashes, render receipts, stale-render gating, and share-page byte verification. Those should be preserved.

The remaining gaps are structural:

- `LightDeck1.3/index.html` currently builds a loose proposal brief, asks for three free-form JSON fields, accepts aliases or JSON embedded in surrounding text, and immediately writes the result into client-facing proposal fields.
- The current proposal prompt says not to invent facts, but there is no claim-to-source binding, locked-field policy, strict schema rejection, or fact-diff before apply.
- The hidden BYOK path stores provider keys with reversible XOR/base64 browser obfuscation and forwards the decrypted key in the request body.
- The visible settings say AI is included while the hidden credits implementation simply grants 50 units in `localStorage`. The UI itself correctly warns that a backend validator is required before public sale.
- Some AI modes still ask a model to choose package prices or default warranty/timeline language. A model must not own those values.
- The current model selector and provider-specific prompts leak runtime implementation details into the product contract.

These are not prompt-quality problems. They are ownership and validation problems.

## Required ownership boundary

```text
                    +---------------------------+
                    |   LightDeck project data  |
                    | canonical fields + hashes |
                    +-------------+-------------+
                                  |
                           JobTruthPacket
                                  |
                 +----------------+----------------+
                 |                |                |
          Managed API       Contractor API   Codex/Claude plugin
                 |                |                |
                 +----------------+----------------+
                                  |
                       typed CandidatePatch
                                  |
                    +-------------v-------------+
                    | LightDeck policy validator |
                    | schema, facts, permissions |
                    +-------------+--------------+
                                  |
                    +-------------v--------------+
                    | deterministic compiler      |
                    | proposal/invoice/PDF/share  |
                    +-------------+--------------+
                                  |
                       hashes + durable receipt
```

Changing providers must not change which facts are authoritative or how totals, deposits, tax, warranties, invoice status, PDFs, or share links are produced.

## Canonical data contract

Every AI task starts from an immutable `JobTruthPacket`. It is built server-side or by the trusted LightDeck client from normalized project state, canonicalized, hashed, and then frozen for that task.

```json
{
  "schema_version": "lightdeck.truth/1",
  "job_id": "job_uuid",
  "job_revision": 17,
  "truth_hash": "sha256-of-canonical-packet-without-this-field",
  "company": {
    "profile_id": "company_profile_uuid",
    "profile_revision": 4,
    "name": "Contractor-configured company name",
    "service_area": null
  },
  "client": {
    "client_id": "client_uuid",
    "name": "Recorded client name",
    "address": "Recorded project address"
  },
  "design": {
    "design_hash": "sha256",
    "source_photo_sha256": "sha256",
    "placements": [
      {
        "placement_id": "fixture_uuid",
        "catalog_item_id": "catalog_uuid",
        "kind": "uplight",
        "x": 0.431,
        "y": 0.782,
        "beam_degrees": 36
      }
    ]
  },
  "commercial": {
    "currency": "USD",
    "packages": [
      {
        "package_id": "package_uuid",
        "line_items": [
          {
            "line_item_id": "line_uuid",
            "catalog_item_id": "catalog_uuid",
            "description": "Configured catalog description",
            "quantity": 8,
            "unit_price_cents": 42500
          }
        ]
      }
    ],
    "deposit_basis_points": 3000,
    "tax_policy_id": "tax_policy_uuid"
  },
  "terms": {
    "warranty_policy_id": "warranty_uuid",
    "payment_terms_id": "terms_uuid",
    "acceptance_text_revision": 3
  },
  "known_facts": [],
  "unknowns": [
    "exact_wire_footage"
  ]
}
```

Rules:

- All currency is integer cents. Percentages are integer basis points.
- A missing fact is `null` or listed in `unknowns`; it is never replaced by a plausible default inside an AI prompt.
- Catalog descriptions, configured defaults, and selected package data carry stable IDs and revisions.
- `truth_hash`, `design_hash`, and source file hashes bind every downstream artifact to the exact input revision.
- The packet sent to a provider contains only fields required by that task.

## AI job contract

LightDeck creates an `AiJobEnvelope` with explicit permissions:

```json
{
  "schema_version": "lightdeck.ai-job/1",
  "ai_job_id": "ai_job_uuid",
  "task": "proposal_opening_copy",
  "truth_hash": "sha256",
  "input_hash": "sha256",
  "allowed_writes": [
    "/draft/project_title",
    "/draft/executive_summary_blocks",
    "/draft/scope_summary_blocks"
  ],
  "forbidden_fact_classes": [
    "identity",
    "money",
    "measurement",
    "legal",
    "payment_state",
    "share_token"
  ],
  "provider_route": "managed_openai",
  "requires_human_apply": true
}
```

The provider returns only a strict `CandidatePatch`:

```json
{
  "schema_version": "lightdeck.ai-result/1",
  "ai_job_id": "ai_job_uuid",
  "status": "candidate",
  "patches": [
    {
      "path": "/draft/executive_summary_blocks/0",
      "value": "A concise paragraph using approved slots.",
      "claim_refs": [
        "/client/name",
        "/design/placements"
      ]
    }
  ],
  "unknowns": [],
  "warnings": []
}
```

No permissive JSON extraction, key aliases, prose wrapped around JSON, or silent fallback is accepted in production. A malformed or unauthorized result is rejected and recorded; it is never partially applied.

## Locked facts

AI may never author or mutate these classes:

- client or company identity and addresses;
- product SKU, catalog identity, fixture quantity, or selected package;
- unit price, discount, subtotal, tax, total, deposit, balance, or payment status;
- warranty duration, legal terms, acceptance language, or financing claims;
- invoice number, proposal token, share URL, payment URL, or signed state;
- measured footage, installation date, permit requirement, or engineering claim unless the exact approved fact exists in the packet;
- whether a render is verified, current, client-visible, or attached to a proposal.

AI may produce bounded candidates for:

- executive-summary prose using approved facts and merge slots;
- homeowner-friendly rewrites of an existing scope without adding facts;
- classification of uploaded notes into known fields or `unknown`;
- fixture placement suggestions marked as suggestions until accepted;
- photo observations tied to visible evidence and confidence;
- render pixels generated from a frozen design manifest;
- consistency-review findings that never mutate records directly.

## Deterministic compilers

The following outputs are LightDeck code paths, not model prompts:

- package line items and all arithmetic;
- estimate, proposal, invoice, receipt, and PDF layout;
- deposit and balance calculations;
- warranty and payment-term insertion from versioned settings;
- share-page payload construction and token creation;
- invoice numbering and payment state;
- fixture inventory/BOM from accepted placements;
- client-facing send copy assembled from approved templates and slots.

The model can draft optional prose fragments. LightDeck composes the final document from the locked packet, approved fragments, and versioned templates.

## Fact validator

Before any candidate can be applied, LightDeck must:

1. Validate the exact JSON Schema with unknown keys rejected.
2. Verify `ai_job_id`, `truth_hash`, schema version, and task match the open job.
3. Verify every patch path is in `allowed_writes`.
4. Verify every `claim_ref` resolves in the frozen packet and is allowed for the task.
5. Reject uncited names, addresses, currency, percentages, dates, counts, measurements, warranty terms, and URLs.
6. Reject facts whose source is null, unknown, stale, or from another tenant.
7. Render merge slots from LightDeck data after validation; the provider never substitutes locked values.
8. Show an explicit diff and require **Apply draft** where human review is required.
9. Save the accepted result as a new project revision, not an in-place invisible mutation.

For high-value prose, use slot-first output. For example, the model may return `{{client.name}}` or `{{commercial.selected_total}}`; LightDeck resolves the slot after validation. Free-typed numbers are forbidden unless the task schema explicitly permits and cites them.

## Render contract

Night rendering is a distinct artifact lane:

1. LightDeck freezes the source photo bytes and hashes them.
2. LightDeck freezes accepted fixture placements, product mode, beam constraints, scene, and protected architecture regions into a `DesignManifest` and hashes it.
3. The provider receives the exact source photo and the minimum render instructions derived from the manifest.
4. The provider returns image bytes only. It does not decide fixture count, location, product, client identity, or proposal facts.
5. LightDeck verifies exact output bytes, architecture preservation, no added structures, expected light near accepted placements, and no unexplained lighting outside allowed influence regions.
6. LightDeck composites a bottom-left professional presentation stamp after verification: `LIGHTDECK CONCEPT RENDERING`, verified status, a short ID derived from the immutable render receipt, and `DESIGN INTENT · NOT FOR CONSTRUCTION`. Unverified previews say `REVIEW REQUIRED` instead of borrowing the verified mark.
7. LightDeck hashes the exact stamped client-visible bytes and records source hash, design hash, raw output hash, presentation output hash, stamp contract/version, provider route, model, verifier results, and timestamps.
8. Any accepted design edit marks the render stale. Proposal publish rejects stale, missing, substituted, or hash-mismatched render bytes.

An AI vision verifier is one signal, not the source of truth. Deterministic pixel registration and protected-region checks should cover architecture changes and unexpected bright regions where practical.

## Provider routes

### 1. LightDeck Managed AI

Recommended default for in-app buttons, phones, and unattended queue reliability.

- LightDeck owns the API account and server-side provider adapter.
- Usage is charged through the existing durable `ai_usage` mechanism, expanded into an auditable entitlement and credit ledger.
- Each task has a server-defined provider/model allowlist, token/image limits, unit cost, timeout, and retry policy.
- Contractors never see or receive LightDeck provider keys.
- A later paid refill flow requires a separately approved Stripe/payment change.

### 2. Contractor BYOK API

Recommended advanced option for contractors who want provider-direct billing.

- OpenAI and Anthropic API billing are separate from ChatGPT and Claude consumer subscriptions.
- Do not store API keys in `localStorage`, IndexedDB, exported backups, logs, analytics, request receipts, or proposal payloads.
- Preferred v1: a LightDeck desktop companion stores the key in the OS keychain and performs provider calls locally.
- Acceptable hosted alternative: a dedicated envelope-encrypted tenant secret vault with narrow decrypt authority, audit logs, redaction, and key deletion/rotation support.
- Remove the existing reversible XOR/base64 storage path before exposing BYOK publicly.

### 3. Use my Codex membership

Recommended subscription-backed experience: **LightDeck for Codex**, a plugin containing a LightDeck skill and OAuth-authenticated MCP app/tools.

Example user request inside Codex:

> Build a draft proposal for the Oak Street job, use the accepted Enhanced package, and show me the fact-check before saving.

The plugin reads a scoped `JobTruthPacket`, generates only the allowed candidate fields, submits the typed candidate to LightDeck, and receives a validation result plus preview URL. The contractor explicitly approves the draft. Credentials for the contractor's ChatGPT/Codex account remain with Codex; LightDeck receives only its own OAuth token and typed tool calls.

This uses the contractor's Codex entitlement in a provider-supported user surface. It does not turn their ChatGPT subscription into a LightDeck API key.

A local `codex exec`/SDK bridge is technically possible and can reuse saved local authentication, but it should remain a private beta until OpenAI confirms the intended commercial use. It must be user-initiated, run on the contractor's trusted computer, use the least sandbox permissions, and never copy `~/.codex/auth.json` into LightDeck. It also cannot guarantee mobile/background execution when that computer is offline.

### 4. Use my Claude membership

The parallel user-controlled experience is a LightDeck MCP server or skill used from Claude Code.

- Claude Pro/Max can authenticate Claude Code locally; Anthropic API usage remains separately billed.
- `claude -p` can return machine-readable output, but a hosted LightDeck service cannot consume the contractor's Pro plan as an Anthropic API quota.
- Use the same `JobTruthPacket`, permissions, validation, and receipt contract.
- Treat an embedded local bridge as experimental; use the Anthropic API for production application automation.

LightDeck should not require one provider. Codex is the first recommended membership integration because it also has a documented SDK/plugin/app path and current image-generation capability. Claude remains an adapter, not a separate product architecture.

## Credits and entitlement ledger

Replace browser credits with server truth before any public credit sale.

Minimum tables:

- `ai_entitlements`: account, plan, included units, reset rule, allowed task classes;
- `ai_ledger`: immutable debit, refund, grant, expiration, source, and idempotency key;
- `ai_jobs`: tenant, task, truth/input hashes, route, status, unit reservation, timestamps;
- `ai_job_results`: schema version, candidate hash, validation decision, accepted revision;
- `ai_artifacts`: source/design/output hashes, storage pointer, visibility, stale state, receipt;
- `ai_devices`: paired local companion public keys and revoked state, if the companion ships.

Required state machine:

```text
created -> reserved -> running -> candidate -> validated -> accepted
   |          |          |            |           |
cancelled   refunded   failed       rejected     stale
```

Reserve units before a managed call. Commit the debit only on the defined billable outcome. Refund idempotently on non-billable failures. A client-provided balance is display-only and never authorizes work.

## Privacy and tenant isolation

- Send only task-required fields to a provider.
- Do not send complete proposal histories when one job revision is enough.
- Keep private client photos out of public demos, reusable fixtures, and tracked test assets.
- Show the contractor which provider route will receive data before the first task.
- Record the provider and applicable consumer/commercial privacy class in the receipt.
- Default client photos and proposals to the commercial API path when predictable business-data handling is required.
- Every lookup, job, artifact, and receipt is account-scoped server-side; client IDs never provide authorization.

## Required acceptance tests

### Fact-lock adversarial suite

For every provider adapter, inject otherwise-valid candidate output containing:

- a different client/company name or address;
- a changed fixture count, product, package, or quantity;
- a fabricated price, discount, total, deposit, tax, or warranty;
- an invented measurement, install date, permit, or performance claim;
- a URL/token or another tenant's fact;
- an extra schema key, aliased key, wrapped JSON, or partial response.

Every case must fail closed, apply no partial mutation, reserve no stale result, and produce a sanitized rejection receipt.

### Compiler suite

- Same frozen packet and compiler version produce identical money, terms, invoice, and PDF fields regardless of provider.
- All totals reconcile from integer cents.
- Settings revisions flow to proposal, share page, invoice, PDF, and receipts.
- Unknown fields render as an explicit review requirement or are omitted; they never receive a guessed default.

### Render suite

- Exact fixture placement and design hashes bind source to output.
- Accepted placement edits make the previous render stale.
- Share bytes hash to the stored presentation artifact.
- Added structures, missing expected light, unexplained bright regions, or protected-edge movement fail verification.
- Real private project proof stays outside tracked public fixtures; tracked e2e uses licensed generic fixtures.

### Credential and route suite

- Provider keys never appear in browser storage, logs, exports, analytics, exceptions, or receipts.
- Managed usage, BYOK usage, and membership-plugin usage have distinct receipts and billing behavior.
- Offline local companions fail honestly and allow the user to choose managed AI or wait; no silent provider switch.
- Revoked devices and expired OAuth tokens cannot read or write jobs.

## Rollout order

### Phase 0: stop hallucination at the current seam

1. Replace permissive proposal-writer parsing with an exact schema and a server-side fact validator.
2. Change the model output to slot-first candidate copy plus claim references.
3. Remove model-owned package prices, warranties, timelines, deposits, and totals from AI modes.
4. Add adversarial red-to-green tests proving locked facts cannot change.
5. Keep all outputs as drafts; preserve the existing human apply/send boundary.

### Phase 1: provider-neutral job runtime

1. Add `JobTruthPacket`, `AiJobEnvelope`, `CandidatePatch`, and receipt modules.
2. Add `ai_jobs`, result, artifact, entitlement, and ledger persistence.
3. Wrap current OpenAI and Anthropic calls behind adapters.
4. Route proposal copy, scope rewrite, photo analysis, placement suggestions, and renders through task-specific contracts.
5. Migrate render provenance into the shared artifact receipt format without weakening existing gates.

### Phase 2: managed and BYOK production paths

1. Make managed AI the default, with honest task costs and server ledger balances.
2. Replace browser BYOK with keychain companion or reviewed encrypted vault.
3. Add provider/data disclosures and route health status.
4. Add owner-approved Stripe refill packs only after the ledger is proven independently.

### Phase 3: membership integrations

1. Ship LightDeck OAuth/MCP tools with read, draft-write, validate, preview, and accept scopes.
2. Package the first LightDeck for Codex plugin and test it against a private account.
3. Add the corresponding Claude Code MCP configuration only if contractor demand justifies it.
4. Evaluate a paired local companion after provider terms and offline/mobile behavior are explicit.

### Phase 4: full LightDeck-owned artifact proof

1. Unify proposal, invoice, PDF, render, and share receipts.
2. Show a compact provenance panel to the contractor.
3. Produce private real-project showcase evidence and generic tracked regression fixtures.
4. Deploy only after full local gates, migration verification, live SHA verification, and live browser retest.

## Go-live gates

Do not claim this architecture is live until all are true:

- no production AI path accepts permissive or aliased JSON;
- no provider can write a locked fact;
- no API key is stored with browser-reversible obfuscation;
- credits and entitlements are server-authoritative;
- proposal, invoice, PDF, share page, and render all bind to frozen truth/artifact hashes;
- a real customer proof uses the authorized private project sources and is not confused with a synthetic house fixture;
- payment changes, migrations, merge, push, and production deployment receive their distinct approvals and evidence;
- deployed SHA, database migrations, API health, and browser journeys are directly verified.

## Official capability boundaries reviewed

- OpenAI Codex supports local ChatGPT subscription authentication and separate API-key authentication: https://learn.chatgpt.com/docs/auth
- `codex exec` supports non-interactive runs and JSON Schema output, and reuses saved local authentication: https://learn.chatgpt.com/docs/non-interactive-mode
- The Codex SDK can programmatically control local Codex threads: https://learn.chatgpt.com/docs/codex-sdk
- Codex image generation uses included plan limits when ChatGPT-authenticated and API pricing when API-key-authenticated: https://learn.chatgpt.com/docs/pricing#image-generation-usage-limits
- OpenAI consumer terms apply to consumer-authenticated Codex and restrict credential sharing and programmatic extraction, so commercial bridge use needs explicit review: https://openai.com/policies/terms-of-use/
- Claude Pro/Max includes local Claude Code usage, while Anthropic API billing remains separate: https://support.anthropic.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan
- Claude Code offers non-interactive JSON output: https://docs.anthropic.com/en/docs/claude-code/cli-usage
- Anthropic distinguishes consumer-plan data handling from commercial/API services: https://www.anthropic.com/news/updates-to-our-consumer-terms
