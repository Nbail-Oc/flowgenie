# Product Vision Document
## AI-Powered n8n Workflow Builder ("Flowmith" — working name)

---

## 1. Vision Statement

Anyone who can describe an automation in plain English should be able to have a working, tested n8n workflow running in their own account within minutes — without learning n8n's node model, writing expressions, or debugging silently-failing HTTP nodes.

We are not another workflow builder. n8n already does that well. We are the **conversational layer** that sits on top of a user's existing n8n instance, turns intent into a working workflow, tests it against real executions, and hands back something that's already been verified to run — not just JSON that *should* work.

---

## 2. The Problem

n8n is powerful but has a steep learning curve:

- Users know *what* they want ("when a Stripe payment fails, notify me on Slack and log it to a sheet") but not which nodes, expressions, or credential setup that requires.
- Building it manually means trial-and-error: add a node, run it, look at raw JSON output, guess the expression syntax, run again.
- Copy-pasted workflows from templates/forums rarely match the user's exact services, field names, or logic.
- There's no "does this actually work" signal until the user manually tests it — often days after activating it, when it silently fails in production.

Existing AI coding assistants can write code, but nobody has built one that closes the loop specifically for n8n: **build → execute → observe real failures → fix → confirm** — inside the user's real environment.

---

## 3. The Solution

A chat-based product where the user describes an automation, the AI asks clarifying questions, proposes a workflow, and — critically — **builds it directly into the user's n8n instance and test-runs it** before saying "done." Failures come back as plain-English explanations, not stack traces, and the AI fixes them itself in the next turn.

### Core loop
1. **Describe** — user explains the automation in natural language.
2. **Clarify** — AI asks only what it needs (trigger type, which service, expected output) rather than a long form.
3. **Build** — AI generates the workflow JSON and creates it in the user's n8n account via the n8n API.
4. **Test** — AI runs a real (or mocked) execution and inspects the result.
5. **Fix** — if something fails, AI diagnoses it in plain English and patches the workflow automatically, re-testing until it passes.
6. **Confirm & activate** — user reviews a visual diff, approves, and the workflow goes live.

---

## 4. Our Core Differentiator: Grounded AI, Not Guessing AI

Most "AI builds your automation" tools are a general-purpose LLM writing JSON from memory and hoping it matches n8n's actual node schemas. That's why they produce workflows with invented parameter names, wrong enum values, and nodes that silently fail on import.

We don't rely on the model's memory or on scraping documentation prose. We ground every generation step in three layers of real, verifiable information, from cheapest/fastest to most authoritative:

1. **Exact node schemas, not descriptions.** Every n8n node's true parameter shape, required fields, and enums are extracted directly from n8n's own code (not the docs website) and matched to the specific n8n version the user is running. Before any JSON is proposed, it's validated against this — instantly, deterministically, no model guessing involved.
2. **Live web knowledge for the judgment calls.** A dedicated research agent pulls current best-practice patterns, service-specific quirks, and community workarounds in real time — so the coding agent isn't limited to stale training data, and can ask for exactly the piece of context it's missing mid-build.
3. **Real execution, in the user's own instance.** The final check is never "the AI thinks this is right" — it's an actual test run against n8n, with real results feeding back into the fix loop.

This is the difference between a tool that *writes plausible-looking automation code* and one that *hands you something that has already been proven to run.* It's also our defensibility: competitors copying the chat UI don't get the grounding pipeline underneath it for free.

---

## 5. Target Users

| Segment | Why they need this |
|---|---|
| Solo founders / indie hackers | Want automations fast, don't have time to learn n8n deeply |
| Ops / marketing / RevOps folks | Know the business logic, not the technical implementation |
| Agencies building client automations | Need to produce working workflows quickly and repeatably |
| Existing n8n users | Want to extend/modify workflows conversationally instead of digging through the canvas |

---

## 6. Product Principles

1. **We build in the user's own n8n instance, not ours.** No hosting, no lock-in, no "export your data" problem. Trust is the whole product.
2. **Never touch real service credentials.** Slack tokens, API keys, OAuth — all stay inside n8n, created and owned by the user. We only ever reference credentials by name.
3. **Nothing ships untested.** A workflow isn't "done" until it has run at least once against sample or real data and produced the expected output.
4. **Show, don't just tell.** Every AI-driven change is visible as a diff on the actual workflow graph before it's applied.
5. **Undo is always available.** Every AI edit is a versioned snapshot; rollback is one click.

---

## 7. Key Features (v1 scope)

- **Conversational workflow builder** — describe → clarify → build, powered by a multi-agent system (coding agent + research agent) rather than a single model guessing in one shot.
- **Schema-grounded generation** — every node is validated against its exact real parameter schema before it's ever written to n8n, eliminating most hallucinated-parameter failures before they happen.
- **Direct n8n integration** — connects to the user's instance via API key, creates/edits workflows live.
- **Visual diff preview** — before/after graph view for every AI-proposed change, rendered on the same node/edge model n8n itself uses.
- **Test-driven build loop** — AI executes the workflow against sample data, reads real n8n execution results, and iterates on failures automatically.
- **Plain-English error translation** — raw n8n/node errors turned into human explanations with a suggested fix.
- **Import & extend existing workflows** — point the AI at an existing workflow and ask it to modify or extend it, with full context of what's already there.
- **Version history & rollback** — every AI-driven change is snapshotted; revert anytime.
- **Cost & complexity warnings** — flags nodes likely to hit rate limits, incur API costs, or need extra setup, before activation.
- **Credential-safe by design** — the app never stores or sees third-party service credentials; it only checks whether a named credential exists in n8n and asks the user to create it there if not.

## 8. Features (v2+ / later)

- **Template library** — users can publish workflows the AI built (with credentials/PII stripped) for others to reuse as starting points.
- **Scheduled / webhook trigger simulation** — let users "fire" a trigger manually from chat to test end-to-end without waiting for the real event.
- **Multi-workflow reasoning** — AI understands relationships across a user's whole n8n instance (e.g. "which workflows touch this Airtable base?").
- **Team collaboration** — shared workspace, comments on workflow versions, approvals before activation.
- **Proactive monitoring** — AI periodically checks execution logs of live workflows and flags recurring failures unprompted.

---

## 9. Non-Goals (explicitly out of scope)

- We are **not** building our own workflow execution engine — n8n remains the runtime, always.
- We are **not** a credential vault or secrets manager.
- We are **not** trying to replace the n8n canvas for power users who prefer to build by hand — this is for people who'd rather describe intent.
- We are **not** initially supporting workflow platforms other than n8n (Zapier, Make, etc.) — depth over breadth for v1.

---

## 10. Success Metrics

- **Time-to-first-working-workflow** (from first message to a passing test execution) — target under 5 minutes for common automations.
- **% of AI-built workflows that pass their first real execution without manual user intervention.**
- **Schema-validation pass rate** — % of AI-generated node configs that pass deterministic schema validation on the first attempt, before ever hitting n8n. A rising trend here is a direct signal the grounding pipeline is working.
- **Retention**: users returning to build a 2nd, 3rd workflow (signal that trust in the "builds correctly in my instance" promise held up).
- **Rollback rate**: how often users need to revert an AI change (lower is better, but non-zero is expected and fine — the safety net matters more than perfection).

---

## 11. Risks & Open Questions

| Risk | Mitigation |
|---|---|
| n8n API changes/breaks compatibility | Pin against a tested API version range; monitor n8n release notes; abstract API calls behind an internal client layer |
| Users on old self-hosted n8n versions with limited API support | Detect version on connect, gracefully degrade features (e.g. disable live test-execution if the instance can't support it) |
| AI generates a workflow that "passes" the test but is semantically wrong | Always require explicit user confirmation before activation; test loop reduces but doesn't eliminate the need for human review |
| Trust concerns around storing n8n API keys | Encrypt at rest, allow key rotation/revocation anytime from n8n's side, be transparent about exactly what scopes are used |
| Complex workflows with many branches are hard to diff visually | Start with node-level diff (added/removed/modified) before attempting full semantic diffing |
| Node schema extraction pipeline falls behind new n8n releases/community nodes | Automate schema extraction to run on new n8n version releases; fall back to live web search + explicit user confirmation for unrecognized node types |

---

## 12. One-Line Pitch

**"Describe your automation. We build it, test it, and hand you back a workflow that already works — live in your own n8n."**