# Technical Design Document (TDD)
## AI-Powered n8n Workflow Builder

---

## 1. Overview

A Next.js application that lets a user connect their own n8n instance, describe automations in a chat interface, and have an AI (Claude) generate, test, and iterate on real n8n workflows via n8n's REST API — with visual diffs, versioning, and rollback built in.

Everything lives in one Next.js codebase for v1. No microservices, no separate backend, no custom execution engine.

---

## 2. Architecture Summary

```
┌───────────────────────────────────────────────────────────────────┐
│                          Next.js App (Vercel)                      │
│                                                                      │
│  ┌────────────┐   ┌───────────────┐   ┌────────────────┐          │
│  │  Chat UI    │   │  Coding Agent  │   │  Diff / Graph  │          │
│  │ (Vercel AI  │──▶│  (Claude,      │──▶│  Viewer        │          │
│  │  SDK)       │   │  orchestrator) │   │  (React Flow)  │          │
│  └────────────┘   └──────┬────────┘   └────────────────┘          │
│                           │  tool call: gather_context()            │
│                           ▼                                         │
│                 ┌────────────────────────┐                          │
│                 │  Context Gatherer Agent  │                        │
│                 │  (Claude, sub-agent)     │                        │
│                 └──┬─────────┬──────────┬─┘                        │
│                    │         │          │                          │
│      ┌─────────────▼──┐ ┌────▼──────┐ ┌─▼────────────────┐         │
│      │ get_node_schema │ │ web_search│ │ search_templates  │         │
│      │ (deterministic, │ │ (Tavily)  │ │ (community n8n    │         │
│      │ from schema DB) │ │           │ │ workflow examples) │         │
│      └─────────────────┘ └───────────┘ └────────────────────┘        │
│                           │                                         │
│                           ▼ generated/edited node JSON               │
│                 ┌────────────────────────┐                          │
│                 │  Deterministic JSON      │  fail → back to        │
│                 │  Schema Validator         │  Coding Agent          │
│                 └───────────┬───────────┘                          │
│                             │ pass                                  │
│                             ▼                                       │
│                 ┌───────────────────┐                               │
│                 │  n8n API Client    │                               │
│                 │  (internal lib)    │                               │
│                 └─────────┬─────────┘                               │
│                           │ HTTPS (API key)                         │
└───────────────────────────┼─────────────────────────────────────────┘
                             ▼
                 ┌───────────────────────┐
                 │  User's n8n instance    │
                 │  (self-hosted or Cloud) │
                 └───────────────────────┘

  Supabase Postgres (via supabase-js) — workflows, versions, templates, node schema cache (auth deferred to Phase 2)
```

---

## 3. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript** | Frontend + backend (API routes / Server Actions) in one project |
| UI | **Tailwind + shadcn/ui** | Fast, consistent, accessible components |
| AI orchestration | **Anthropic SDK (Claude) + Vercel AI SDK** | Streaming chat, tool-calling, works natively with Next.js; used for both the Coding Agent and Context Gatherer sub-agent |
| Live web search (grounding) | **Tavily API** | LLM-optimized search results, zero infra to run for v1; used only for behavioral/pattern knowledge, not for node schemas |
| Node schema source (grounding) | **Self-hosted schema extraction pipeline** (inspired by open-source `n8n-schema-generator` approach) | Extracts exact parameter schemas, enums, and required fields directly from n8n's own node code per version — deterministic ground truth, not doc prose |
| Schema validation | **Ajv (JSON Schema validator)** | Fast, deterministic validation of AI-generated node JSON against extracted schemas, before any n8n API call |
| Workflow graph rendering | **React Flow (`@xyflow/react`)** | Same library n8n's own canvas uses; ideal for rendering workflow JSON and diffs |
| Database | **Supabase Postgres, accessed via `supabase-js`** | No raw connection strings or pooler config to manage — reads/writes go through Supabase's REST client (project URL + API key), same as any SaaS SDK |
| LLM providers (multi-provider) | **Anthropic (Claude, primary)** + configurable alternates: **GroqCloud, OpenRouter, AI/ML API** | Claude drives the Coding Agent and Context Gatherer by default; the provider layer is abstracted so faster/cheaper models can be swapped in for specific sub-tasks or as fallback (see Section 7.6) |
| Auth | *(deferred — not in v1, see Section 10)* | Ship single-tenant/no-auth first; add Clerk or Supabase Auth once multi-user access is actually needed |
| Background jobs (post-MVP) | **Trigger.dev or Inngest** | For long test-execution polling at scale |
| Hosting | **Vercel** | Zero-config Next.js deploys, serverless API routes |
| Secrets encryption | AES-256-GCM with server-side key (Vercel encrypted env var) or Supabase Vault | For encrypting stored n8n API keys |

---

## 4. Data Model (initial schema, simplified)

Tables live in Supabase Postgres, created via the Supabase dashboard's SQL editor or Table Editor — no ORM/migration tool required for v1. `supabase-js` is used for all reads/writes from the Next.js app.

```sql
create table users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  created_at timestamptz default now()
);

create table n8n_connections (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id),
  instance_url text not null,
  api_key_encrypted text not null,
  n8n_version text,
  created_at timestamptz default now()
);

create table tracked_workflows (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id),
  n8n_workflow_id text not null,  -- ID inside the user's n8n instance
  name text not null,
  created_at timestamptz default now()
);

create table workflow_versions (
  id uuid primary key default gen_random_uuid(),
  tracked_workflow_id uuid references tracked_workflows(id),
  workflow_json jsonb not null,      -- full snapshot at this point
  change_summary text,               -- AI-generated plain-English summary of the diff
  test_result jsonb,                 -- last execution result at this version
  created_at timestamptz default now()
);

create table templates (
  id uuid primary key default gen_random_uuid(),
  author_id uuid references users(id),
  title text not null,
  description text,
  workflow_json jsonb not null,      -- credentials/PII stripped
  published_at timestamptz
);

create table node_schemas (
  id uuid primary key default gen_random_uuid(),
  n8n_version text not null,         -- schemas are version-scoped
  node_type text not null,           -- e.g. "n8n-nodes-base.slack"
  schema_json jsonb not null,        -- extracted parameter schema, required fields, enums, defaults
  extracted_at timestamptz default now(),
  unique (n8n_version, node_type)
);

create table credential_schemas (
  id uuid primary key default gen_random_uuid(),
  n8n_version text not null,
  credential_type text not null,     -- e.g. "slackApi"
  schema_json jsonb not null,
  extracted_at timestamptz default now(),
  unique (n8n_version, credential_type)
);
```

Example query via `supabase-js`:

```javascript
const { data, error } = await supabase
  .from('workflow_versions')
  .insert({
    tracked_workflow_id: workflowId,
    workflow_json: newJson,
    change_summary: summary
  });
```

---

## 5. Core Flows

### 5.1 Connecting an n8n instance
1. User enters instance URL + API key (generated in n8n: Settings → API).
2. Backend validates by calling `GET /api/v1/workflows` (or `/rest/workflows` for older versions) — confirms key works and detects n8n version.
3. API key is encrypted (AES-256-GCM) and stored in `n8n_connections`.
4. Nothing beyond the API key ever leaves the user's control — no service credentials are requested or stored.

### 5.2 Conversational build loop
1. User describes the automation in chat.
2. Coding Agent (Claude) asks clarifying questions only where genuinely ambiguous (trigger type, target service, expected data shape) — not a rigid form.
3. Coding Agent calls the Context Gatherer sub-agent for any node/service it needs grounding on (see Section 7 — AI Grounding & Multi-Agent Architecture) and generates workflow JSON (nodes + connections).
4. **Deterministic schema validation** — the generated JSON is validated against the extracted `node_schemas`/`credential_schemas` records for the user's n8n version, using Ajv. Failures are returned to the Coding Agent immediately, with no n8n API call spent — this catches the large majority of structural errors before they ever reach the user's instance.
5. Once schema-valid, backend calls n8n API to create the workflow (inactive).
6. Backend triggers a test execution:
   - If the workflow starts with a manual/webhook trigger, call the manual execution / webhook-test endpoint with sample or pinned data.
   - If it depends on credentials the user hasn't set up yet, the AI stubs that branch, tells the user what credential is needed, and continues testing the rest.
7. Backend fetches the execution result (`GET /executions/:id` or equivalent) — per-node output and error data.
8. If failed: Claude receives the raw error + node context, translates it to plain English, optionally calls the Context Gatherer again for more information, patches the workflow JSON, and the loop repeats (steps 4–8) up to a bounded retry count (e.g. 5 iterations) before surfacing to the user for manual input.
9. If passed: user is shown a summary + diff view and asked to confirm activation.

### 5.3 Diff preview
1. Before applying any AI-proposed change, compute a structural diff between the previous `workflow_versions.workflow_json` and the proposed new JSON: added nodes, removed nodes, modified parameters/connections.
2. Render both states in React Flow, with added nodes highlighted green, removed red, modified amber — using the same node types/icons n8n uses for visual familiarity.
3. User approves or rejects; only on approval does the backend push the change to n8n and write a new `workflow_versions`.

### 5.4 Version history & rollback
- Every applied change creates a new `workflow_versions` row with the full JSON snapshot.
- Rollback = fetch a prior version's JSON, PUT it back to n8n via the API, create a new version row documenting the rollback.

### 5.5 Import existing workflow
- User selects a workflow that already exists in their n8n instance (`GET /workflows/:id`).
- It's imported as a baseline `tracked_workflows` + initial `workflow_versions`.
- The AI is given the full JSON as context for any follow-up conversational edits.

### 5.6 Cost/complexity warnings
- Before activation, run a static check across the proposed nodes:
  - Flag nodes known for rate limits or paid usage tiers (HTTP Request to metered APIs, certain integrations).
  - Flag loops/batches that could cause many repeated calls.
  - Present as a short pre-activation checklist, not a blocker.

### 5.7 Template publishing (v2)
- User opts to publish a `tracked_workflows` version as a `templates`.
- Backend strips all credential references and replaces literal values that look like secrets/PII with placeholders (basic pattern matching + AI review pass).
- Published templates are browsable/searchable and can be one-click imported by other users (still requires them to attach their own credentials).

---

## 6. n8n API Integration Notes

- Use n8n's **public REST API** (`/api/v1/...`) where available for creating/reading/updating workflows — this is the stable, documented surface.
- Some capabilities (like triggering a manual test execution) may only be available via internal endpoints (`/rest/workflows/run`) depending on n8n version — abstract this behind an internal `N8nClient` class so version differences are isolated to one place, and degrade gracefully (e.g., "test execution isn't supported on your n8n version — here's the workflow, please test manually") when unavailable.
- Detect and store the connected instance's n8n version on connect; feature-flag test-execution and other version-sensitive features accordingly.
- All calls authenticated via the user's own API key — the app has no elevated access beyond what the user's key grants.

---

## 7. AI Grounding & Multi-Agent Architecture

This is the core technical differentiator: instead of a single LLM writing workflow JSON from memory, generation is split across two cooperating agents plus a deterministic validation layer, so the model is never guessing at things that have a knowable, exact answer.

### 7.1 Why not RAG over scraped docs?

Scraping the n8n docs site (e.g. with Crawl4AI) and embedding it for retrieval was the first idea considered, but it has two problems:
- **Staleness** — requires a re-crawl/re-embed pipeline to stay current with n8n releases.
- **Wrong granularity** — docs describe node behavior in prose; they're not a reliable source for the *exact* JSON shape a node expects (field names, enum values, nested structures). Prose-derived answers still leave the model guessing at the literal JSON.

Instead, grounding is split by what kind of knowledge is actually needed:

| Knowledge type | Source | Why |
|---|---|---|
| Exact node parameter schema, enums, required fields | **Extracted schema DB** (`node_schemas`/`credential_schemas`, keyed by n8n version) | Deterministic ground truth from n8n's own node code — not guessable, not worth searching for |
| Behavioral nuance, service quirks, current best practice | **Live web search (Tavily)** | Changes over time, not exactly reducible to a schema, benefits from real-time freshness |
| Realistic example patterns for a given automation type | **Community template search** | Grounds structure/composition (which nodes are typically chained together) without hallucinating a pattern from scratch |
| Whether the actual workflow runs | **Real n8n execution** | The only test that matters in the end |

### 7.2 Agent roles

**Coding Agent (orchestrator)**
- Owns the conversation with the user.
- Decides what workflow to build/edit.
- Calls the Context Gatherer as a tool whenever it needs grounding it doesn't already have in context — mid-generation, not just once up front. This keeps context usage lean versus dumping full documentation into every turn.
- Emits workflow JSON, receives validation/execution feedback, and iterates.

**Context Gatherer Agent (sub-agent)**
- Receives a specific, scoped request from the Coding Agent (e.g. "what's the exact schema for `n8n-nodes-base.slack` v2.3?" or "how do people typically deduplicate Stripe webhook events in n8n?").
- Has three tools available and chooses between them based on the type of question:
  - `get_node_schema(nodeType, n8nVersion)` — deterministic DB lookup, always tried first for anything schema-shaped.
  - `web_search(query)` (Tavily) — for behavior, patterns, service-specific edge cases.
  - `search_community_templates(query)` — for realistic example compositions.
- Returns a compact, structured summary back to the Coding Agent — not raw search results or full doc pages.

### 7.3 Node schema extraction pipeline

- A background job (not user-facing) periodically extracts parameter schemas for all built-in n8n nodes and credential types, per n8n version, and writes them into `node_schemas` / `credential_schemas`.
- Approach: load the actual n8n node package for a given version, instantiate/reflect over each node class the way n8n itself does internally (via `NodeHelpers.getNodeParameters()`-equivalent logic), and serialize the resulting schema, defaults, and validation rules to JSON. This mirrors the approach used by existing open-source tooling in this space and should be run in CI whenever a new n8n version is detected.
- Schemas are looked up by the **specific version the connected user's instance is running**, not a generic "latest" — this avoids generating JSON for parameters that don't exist in the user's actual n8n version.
- For self-hosted users on older versions, or for community (non-built-in) nodes without an extracted schema, the Context Gatherer falls back to `web_search` and flags the result to the user as lower-confidence, encouraging manual review before activation.

### 7.4 Deterministic validation layer

- Before any workflow JSON is sent to the n8n API, it's validated with **Ajv** against the relevant `node_schemas` entries: required fields present, enums valid, types correct, no unknown parameters.
- This step is fast (no network call, no LLM call) and catches the majority of structural errors immediately, well before the more expensive real-execution test loop in Section 5.2.
- Validation failures are returned to the Coding Agent as structured errors (which field, what was expected) rather than free text, so the fix is precise rather than another guess.

### 7.5 Search tool choice: Tavily vs SearxNG

| | Tavily | SearxNG |
|---|---|---|
| Setup | Managed API, zero infra | Self-hosted instance to run and maintain |
| Result quality | Pre-cleaned, LLM-optimized snippets | Raw aggregated results, needs more filtering |
| Cost model | Pay-per-search | Free, but hosting/maintenance cost instead |
| Reliability | Managed uptime | Own responsibility |

**Decision for v1: Tavily.** Search volume is targeted (schema questions are handled by the deterministic layer, not search), so cost stays low, and it removes an entire piece of infrastructure to run and monitor. Revisit SearxNG as a cost optimization once search volume is well understood in production.

### 7.6 LLM provider strategy

Rather than hardcoding a single provider, the AI orchestration layer sits behind a provider abstraction so models can be swapped per-agent or per-task without touching the Coding Agent's or Context Gatherer's logic:

- **Anthropic (Claude) — primary.** Drives the Coding Agent (main reasoning, JSON generation, error diagnosis) and the Context Gatherer by default. This is the quality bar the product is built around.
- **GroqCloud — fast inference alternate.** Useful for latency-sensitive, low-complexity sub-tasks (e.g. quick classification of "does this question need a schema lookup or a web search," summarizing search results before they go back to the Coding Agent) where raw speed matters more than deep reasoning.
- **OpenRouter — unified fallback/redundancy.** Single API surface to a wide range of models; useful as a fallback if the primary provider has an outage, or for A/B testing whether a cheaper model handles a given sub-task well enough.
- **AI/ML API — additional fallback/cost option.** Same rationale as OpenRouter — another aggregator to route around outages or optimize cost on non-critical sub-tasks.

Implementation: build a thin internal `LLMClient` interface (`complete(prompt, options)`) with provider-specific adapters underneath — the Vercel AI SDK already supports multiple providers behind one interface, which is the natural fit here. Model/provider selection can be config-driven per agent role (e.g. `CODING_AGENT_PROVIDER=anthropic`, `CLASSIFIER_PROVIDER=groq`) so it's changeable without a redeploy of core logic. Keep the actual **workflow-generation and error-fixing reasoning on Claude** for v1 — that's the part where reasoning quality directly affects whether the product's core promise ("hands you back something that already works") holds up; treat the other providers as levers for cost/speed on secondary tasks, not replacements for the primary reasoning path.

---

## 8. Security Considerations

- n8n API keys encrypted at rest (AES-256-GCM), decrypted only in-memory per request.
- No third-party service credentials (Slack tokens, OAuth secrets, etc.) are ever requested, transmitted through, or stored by the app — these remain entirely inside the user's n8n instance.
- Users can revoke access anytime by rotating/deleting the API key in n8n.
- Rate-limit and scope the app's own API routes to prevent abuse of a connected instance.
- Sample/test data used in test executions should avoid touching real destination systems where possible (e.g. prefer pinned/mock data over live sends during the build loop; only "real" execution happens on explicit user confirmation).

---

## 9. Testing Strategy (for the product itself, not just generated workflows)

- **Unit tests**: n8n API client wrapper, diff computation logic, JSON validators, Ajv schema validation against known-good and known-bad node configs.
- **Integration tests**: spin up a local/dev n8n instance (Docker) in CI, run the full build → validate → test → fix loop against known workflow specs, assert on outcomes.
- **Schema extraction tests**: run the extraction pipeline against a pinned n8n version in CI, diff output against a committed baseline to catch unexpected breakage.
- **Snapshot tests**: for the diff renderer, given two workflow JSONs, assert the computed diff matches expected added/removed/modified sets.
- **E2E (Playwright)**: chat → workflow created → visible in diff view → activate → appears correctly in a real n8n test instance.

---

## 10. Build Phases

**Phase 1 — MVP**
- Connect n8n instance, conversational build, node schema extraction pipeline (built-in nodes only, single pinned n8n version to start), deterministic schema validation, create workflow via API, basic diff preview, manual test trigger, plain-English error translation, version history. **No auth in v1** — single-tenant/local use is enough to validate the core loop; every `user_id` field can point at one seeded row until real auth is needed.

**Phase 2**
- Context Gatherer sub-agent with Tavily web search + community template search, full automated test-fix loop (bounded retries), multi-version schema extraction (support several recent n8n versions), cost/complexity warnings, import & extend existing workflows. **Add real auth here** (Clerk or Supabase Auth) once the product needs more than one real user account.

**Phase 3**
- Template library & publishing, team/workspace features, proactive monitoring of live workflow failures, community node schema support.

---

## 11. Open Technical Questions

- Exact set of n8n API/version combinations we commit to supporting at launch (affects how much of the test-execution feature is available out of the gate, and how many versions the schema extraction pipeline needs to cover).
- How much of the diff should be "semantic" (e.g. recognizing a renamed field vs. a genuinely new one) vs. purely structural for v1 — recommend starting structural-only and revisiting.
- Whether pinned/mock data for testing should be AI-generated per node or require minimal user input for realistic shapes (likely a hybrid: AI proposes, user can edit).
- How to handle community (non-built-in) nodes that have no extracted schema — web-search fallback with a lower-confidence flag, or exclude them from AI-driven generation entirely for v1.
- Whether the Context Gatherer should cache its own web search results (e.g. per node type) to avoid repeated identical searches across users, and if so, how to keep that cache fresh.
- Exactly when to introduce auth — the moment more than one real person needs isolated data is the trigger; worth deciding this threshold explicitly rather than drifting past it.
- Which specific sub-tasks (if any) actually get routed to GroqCloud/OpenRouter/AI-ML API in practice — start with all traffic on Claude and only split off a sub-task once there's a measured latency or cost reason to.