---
name: preuve-agent-api
description: >-
  Validate startup and business ideas against live market evidence through the
  Preuve AI MCP server. Use whenever the user wants to score or stress-test an
  idea, compare several at once, check founder fit or proof of demand, generate
  new ideas, export validation data, or run a client project in a Consultant or
  Agency workspace — even if they never say "Preuve", and whenever the
  start_analysis / get_agency tools are available. Covers which of the two
  workspaces a request belongs to, and which calls spend money.
---

# Preuve Agent API (MCP)

Preuve AI analyzes startup ideas against live market evidence (60+ sources) and returns a scored verdict with risks, competitors, and citations. This skill is the operating manual for its MCP server. The tools are thin wrappers over an async pipeline: analyses take minutes, results are fetched in stages, and one of the two scan types costs real money. Follow the workflows below and you will never burn quota by accident or hit an avoidable 409.

## Setup (once)

Two transports, same six tools. Both need an API key created at https://preuve.ai (Account → API Keys; shown once at creation) — except the claude.ai connector, which handles the key via OAuth.

**Remote (preferred — no file to install):**

```sh
claude mcp add preuve --transport http https://mcp.preuve.ai/mcp \
  --header "Authorization: Bearer prv_..."
```

In claude.ai (web/desktop), add a custom connector pointing at `https://mcp.preuve.ai/mcp` instead — OAuth sign-in and a consent screen replace manual key handling.

**Local stdio (the original server, still supported):**

```sh
claude mcp add preuve \
  --env PREUVE_API_KEY=prv_... \
  -- node /path/to/preuve-mcp-server.mjs
```

**Doing client work? Ask for the Agency scopes AT CREATION.** There are two, they are enforced per route and neither implies the other: starting a client project needs `agency:write`, while `get_agency` and the two agency reads need `agency:read`. Ask for both. A key never gets either by default — a key created with no preference carries the five personal scopes and nothing else, because these two reach a whole workspace of other people's client reports. There is no way to widen a key afterwards, so a key made without them means making another one.

- **Creating a key by hand**: tick **Include Agency workspace access** in Account → API Keys. Self-serve on a paid personal plan, or in a Consultant or Agency workspace with an active subscription.
- **claude.ai connector**: approve the Agency permissions on the consent screen. Works on any plan, free included. Check the screen actually lists them before approving — if it does not, the client re-sent a scope request it cached at registration, and approving mints another key without them while revoking the one you have.

Skip this on a personal account: the scopes do nothing without a workspace, and the five personal tools need none of them.

One remote-only caveat: an `enrich_analysis` call that starts `founderFit` may time out at ~95s while generation continues server-side; the money is safe (the run is locked), poll `get_analysis` until the module reads `completed`.

## Two workspaces, and how to tell them apart

Five tools act on **your own** ideas. The same verbs act on **client** projects inside a Consultant or Agency workspace once you pass `workspace: "agency"`, and that is a different product with different money:

| Workspace | Calls | Spends |
| --- | --- | --- |
| Personal (default) | `start_analysis`, `get_analysis`, `enrich_analysis`, `export_analysis`, `generate_ideas` | the account's own scan quota (deep only) |
| Agency | `get_agency`, plus `workspace: "agency"` on `start_analysis`, `get_analysis` and `export_analysis` | the workspace's **shared** project credits |

**Default to the personal workspace.** Omitting `workspace` means personal. Only pass `workspace: "agency"`, or call `get_agency`, when the user is talking about a CLIENT: "run this for my client", "add it to the workspace", "list the projects". "Validate my idea" is always a plain `start_analysis`, even on a Consultant account. Picking wrong costs a shared credit that belongs to the whole workspace, or a refusal that wastes a turn.

All six tools appear in `tools/list` for every account, including personal ones with no workspace at all. Seeing `get_agency`, or the `workspace` argument, does not mean the account can use them.

**Two refusals, and they answer different questions.** `403 INSUFFICIENT_SCOPE` is about the KEY: it lacks the scope, and a consultant with a legacy key gets the identical 403. `404 AGENCY_NOT_FOUND` is about the ACCOUNT: the scope is there, the workspace is not. Only the 404 tells you anything about account shape.

Neither one changes what the WORK is, and that is what picks the workspace. Client work stays on `workspace: "agency"` in both cases: on a 403 the user gets the scope (Setup above), and on a 404 they restore workspace access, which is a membership problem no key can fix. Stop and ask, rather than falling back. A personal `start_analysis` on a client's idea stores it outside the workspace and spends the caller's personal quota, so it is the wrong answer to both refusals and there is no undo. Drop `workspace: "agency"` only when the idea is the USER'S OWN.

## The three rules

1. **`scanType: "starter"` is the default, always.** `"deep"` spends a paid scan/token from the account's quota. Only send `"deep"` when the user explicitly asked for a deep/full/paid analysis — if in doubt, ask them first. There is no undo.
2. **Nothing is synchronous.** `start_analysis` returns immediately with `status: "PROCESSING"`. Poll `get_analysis` every 20–30 seconds. Analyses typically take a few minutes (deep runs longer); if one is still processing after ~15 minutes, stop polling, hand the user the `reportUrl` so they can watch it land, and offer to check back — the run keeps going server-side. Hold that 20–30s cadence: create/enrich routes are rate-limited (10/min) and there is a daily per-key ceiling.
3. **Export has a gate.** `export_analysis` on a run that isn't `COMPLETED` returns `409 ANALYSIS_NOT_COMPLETE`; completed but `readyForExport: false` returns `409 ENRICHMENT_NOT_COMPLETE`. The fix for the second is always the same: call `enrich_analysis`, poll again, then export. Read both as sequencing signals rather than failures, and export once `readyForExport` is true: a non-ready analysis is always a 409, never partial data.

## Single analysis, start to finish

1. `start_analysis` with a unique `clientRunId` (e.g. `myagent-2026-07-16-001`), `scanType: "starter"`, the `idea` in plain language, and optionally `targetMarket` / `targetCountry`. Leave `publish` off unless the user wants a public share link — publishing creates a publicly reachable URL.
2. Poll `get_analysis` with the returned `id` until `status` is `COMPLETED` or `FAILED`. Note `get_analysis` never returns report content — only status, `readyForExport`, enrichment/module progress, and URLs. All content comes from the export.
3. If `readyForExport` is `false`, call `enrich_analysis` (idempotent — it only generates what's missing), then poll until `readyForExport: true`.
4. `export_analysis` → structured `ideas-json` v2: `verdict` (GO/NO-GO), `score` (0–100), `risk`, `evidence`, competitors, and `details.citations` with real source URLs. Deep reports additionally carry `details.sections` (business model, go-to-market, Porter forces, VC scorecard, financial projections, …), `details.pivots`, and any generated module payloads.

The `clientRunId` is your idempotency handle: re-sending the same one replays the existing run instead of creating a duplicate. Pick a fresh one per genuinely new analysis. One namespace covers BOTH workspaces on a key, so an id already used for a client project cannot start a personal scan, or the reverse: that answers `409 CLIENT_RUN_ID_CONFLICT`. Flipping `workspace` on a start you have already sent needs a new id, not the old one.

## Reading in-flight status correctly

A deep run reports `analysisTier: "basic"` **while it is still processing** — the tier only flips to `"advanced"` when the deep sections land. This is not a downgrade and not an error. `scanType` reflects what you requested and is reliable from the moment of creation; `analysisTier` means "is the deep result ready yet". A genuine refusal to run deep is always an explicit error (`402 INSUFFICIENT_TOKENS` or a service-disabled error), never a silent starter.

## Deep modules (paid reports only)

`enrich_analysis` accepts `modules: ["proofOfDemand" | "founderFit" | "playbook" | "trends" | "community"]` on a **completed deep** analysis. On a starter/basic report the whole modules request fails `403 MODULES_REQUIRE_DEEP`.

- **Cap: one successful generation per module per report.** Regen attempts return `409 MODULE_ALREADY_GENERATED`. A *failed* generation does not consume the cap — retry it.
- **`founderFit` needs a `founderProfile`** in the same call (hours/week, runway, domain experience, shipped-before, audience, team status). It runs inline — the call can take up to ~90s and returns the completed state directly. If it returns `503 INSUFFICIENT_TIME_BUDGET`, just call enrich again: core enrichment ate the clock on the first call and the retry has full budget.
- **`proofOfDemand`, `playbook`, `trends`, `community` run async**: the call returns `202` with `state: "generating"`; poll `get_analysis` and watch its `modules` map (`not_generated | generating | failed | completed`). If the founderFit call drops mid-flight, read that same `modules` map first and re-call only what it reports as `not_generated` or `failed`. `community` fetches real forum/Reddit/X discussions about the market (~90s) — the analysis pipeline never gathers these on its own, so it's the only way to get them.
- Module payloads then appear in the export under `details.founderFit` / `details.playbook` / `details.proofOfDemand` / `details.communityDemand`.

Ask for modules in the enrich call only when the user wants them — each is extra generation work on their account.

## Generating ideas (`generate_ideas`)

The one tool with no run id: it generates startup ideas synchronously from `interests` (3–500 chars; optional `budget` / `audience` / `industry` / `language`) and spends **no scan quota or tokens** — its budget is separate.

- **Free accounts: 5 generations per rolling 24 hours.** The 6th returns `429 GENERATION_LIMIT` with the limit and an upgrade offer — relay that body to the user as-is; there is no Retry-After to honor because the window is rolling. Paid accounts have no cap (`limit: null`).
- **`pro: true` is a paid perk.** On a free account it is refused with `403 PRO_PAID_REQUIRED` and an upsell payload, consuming nothing. Relay the offer as the answer; a plain retry returns the same refusal.
- **`fit: true` is a paid perk too**: it tailors to the founder profile saved on preuve.ai, and fails soft — `fit: false` in the response means no paid plan OR no saved profile, not an error. On a paid account, suggest completing the profile at preuve.ai; on a free account, completing the profile leaves `fit` false, because the unlock is a paid plan.
- On a free account each idea's `painPoint` comes back locked (`painPointLocked: true`) — that's the tease, not missing data.
- The call is synchronous (up to ~90s on `pro`). If it times out or fails, retry once: failed generations are refunded to the free quota. Follow up a favorite idea with `start_analysis` (`scanType: "starter"` is free).

## Screening several ideas

There is no batch tool. To compare or screen a list, run the single-analysis loop once per idea:

1. `start_analysis` per idea, each with its **own unique** `clientRunId`. Keep everything `"starter"` unless deep is intentional per idea, and give every deep run its `stage` and `budget`.
2. Poll each `get_analysis` until terminal. A completed run with `readyForExport: false` needs `enrich_analysis` on its own run id.
3. `export_analysis` each completed run, then rank them yourself.

The account holds 3 starter and 2 deep analyses in flight at a time. Past that, `start_analysis` answers `429 CONCURRENT_LIMIT_REACHED`, which is pacing, not failure: wait for a run to finish, then retry that idea with the **same** `clientRunId`. Tell the user which ideas you did not manage to score. A ranking that silently covers 7 of 10 ideas misleads them.


## Client projects (Agency workspaces)

For Consultant, Program and AppSumo Agency accounts. Same pipeline, different ledger: every start spends **one shared project credit** from the workspace, never the caller's personal quota, and there is no fallback between the two.

1. **`get_agency` first, always.** One call returns the workspace, your role, the remaining project quota, and the client reports already in it. Call it before the first start of a session so you never burn a shared credit blind.
2. `start_analysis` with `workspace: "agency"`, a unique `clientRunId` and the client's `idea` (40 chars minimum). Always `scanType: "deep"` — there is no starter tier here — so confirm with the user before the first one. Optional `targetMarket` / `targetCountry` / `stage` / `budget` / `language`; send `stage` and `budget` only as the user stated them.
3. Poll `get_analysis` with `workspace: "agency"` and the returned `reportId`, honouring `pollAfterSeconds`. Client reports need no `enrich_analysis`: they are done when `status` is `COMPLETED`.
4. `export_analysis` with `workspace: "agency"` for the content, `verbosity: "summary"` when you only need the headline.

The `reports` array `get_agency` returns covers work already in the workspace, **including reports created on the website**, not just via MCP. Use it to answer "what did we run for this client" without starting anything; page through it with `limit` and `offset`.

On a timeout, re-send the **same** `clientRunId`: it recovers the original project instead of charging a second credit. A different id is a new paid project.

A client project cannot publish a share link or email the client, by design. Client data is confidential — do not send it anywhere the user did not ask for.

## Errors and retries

Every tool error returns the API's JSON error payload as text (never a crash). React by `code`:

| Code / status | Meaning | What to do |
| --- | --- | --- |
| `401` | Missing, malformed, unknown or revoked key | Not retryable from the agent side; tell the user to check their API key |
| `402 INSUFFICIENT_TOKENS` | Deep scan requested, no quota | Tell the user; offer a starter scan instead |
| `403 MODULES_REQUIRE_DEEP` | Modules on a non-deep report | Only offer modules on deep runs |
| `403 INSUFFICIENT_SCOPE` | The KEY lacks the scope, and nothing more | Not retryable. Keep the workspace the REQUEST calls for and get the user the scope: see **Two workspaces**, then **Setup**. Both transports return that recovery in the refusal text — relay it |
| `404 AGENCY_NOT_FOUND` | The scope is present; the ACCOUNT belongs to no workspace, or membership was removed | The only refusal that establishes account shape, and no key fixes it. On the user's OWN idea, drop `workspace: "agency"` and say why. On CLIENT work, stop: ask them to restore workspace access. See **Two workspaces** |
| `409 ANALYSIS_NOT_COMPLETE` / `ENRICHMENT_NOT_COMPLETE` | Sequencing, not failure | Poll / enrich, then retry the export |
| `409 MODULE_ALREADY_GENERATED` | Module cap hit | The payload already exists — just export |
| `409 CLIENT_RUN_ID_CONFLICT` | The id already belongs to a run in the OTHER workspace; one namespace covers both on a key | The run is not lost, and the 409 hands you its id (`conflictingRunId`, or the client `reportId`). READ it with `get_analysis` on that id, plus `workspace: "agency"` if it is a client report. Do NOT rebuild a start body from the 409: you would have to invent an `idea`, and the only one in hand is the other workspace's, which a run that failed before dispatching would then re-run under the original id. To genuinely re-run a failed personal scan, resend YOUR original request for that `clientRunId`, unchanged. For the work you were actually starting, take a NEW `clientRunId` |
| `422 ANALYSIS_FAILED` | The run itself failed | See retry rule below |
| `429 CONCURRENT_LIMIT_REACHED` | Too many analyses in flight for this account | Wait for in-flight runs to finish, then retry the same `clientRunId` |
| `429 DAILY_LIMIT_REACHED` | Per-key daily run ceiling | Stop creating runs today; polling/export still work |
| `429` (route limiter) | >10 creates/min | Back off; slow the loop |

**Retry rule for failed runs:** a `FAILED` run that never attached a report (pre-dispatch failures like `RATE_LIMITED`, `INSUFFICIENT_TOKENS`, `TRIGGER_DISPATCH_FAILED`) is retryable — re-send the **same** `clientRunId` and the run resets and re-dispatches, with anything it claimed already refunded. Runs that did attach a report are strictly idempotent: the same `clientRunId` always returns the stored outcome, so a new attempt needs a new id. Replay the original `clientRunId` first; a fresh one starts, and can bill, a second run.

## Reporting results to the user

- Lead with `verdict` and `score`, then the top risk and the evidence quote — that's the analysis's own summary hierarchy.
- Always give the `reportUrl` (the user's authenticated report view). Only surface `shareUrl` if one exists or the user asked to publish.
- On starter exports, deep-only fields (`details.sections`, `details.pivots`, module payloads) are `null` by design. Present them as locked by tier, and mention a deep scan unlocks them if relevant.
- Cite from `details.citations` when the user asks "based on what?" — every entry is a real research source with a URL.
- Check `idea.inferredContext`: when the user's idea description was short, the analysis filled in assumptions (pricing, delivery model, target segment) and the numbers depend on them. If it's non-null, surface those assumptions to the user — a wrong guess means they should refine the idea text and run a fresh analysis.
