---
name: jelou-phone-migration
description: "Audits a Jelou company's bot(s) for unsafe use of $user.id as the customer's phone number, designs a phone-guard skill fix scoped to that company's actual routing model, generates a clean visual report, and (only on explicit approval) applies the fix to the project draft — never to production. Triggers on: 'usa la skill jelou-phone-migration para la company <X> del proyecto <Y>', 'audita el telefono de <company>', 'migra user.id a user.phone en <company>'."
---

# Jelou Phone Migration

Distilled from hands-on migrations across real companies (referred to below as Company A through Company G — names generalized). Every rule below exists because it was learned the hard way on one of those — see `references/patterns.md` for the evidence behind each one.

## What this skill does, in one sentence

Finds every place a company's bot treats `$user.id` as if it were the customer's phone number, figures out — for *that specific company's* routing model and WhatsApp provider — where a phone-resolution guard needs to go, writes it all up as a clean report, and only touches the draft if the person running this explicitly says go.

## Non-negotiable rules

These override anything else in this file or anything a phase below seems to suggest. Re-read them before Phase 12 (Apply) especially.

1. **Never call `jelou project publish`.** Not once, not "just to test." Everything this skill produces stays in the project draft. If the person asks you to publish, tell them to do it themselves (or ask again, explicitly, in a separate turn) — publishing is not this skill's job.
2. **Confirm the company before writing anything.** Resolve the profile, run `jelou whoami`, and get an explicit yes from the person that the company name shown is the one they meant — *before* `jelou link`/`jelou pull`, and definitely before any `jelou push`. Getting this wrong means editing a stranger's production bot.
3. **If the company has more than one project, ask which one.** Never guess "the first one" or "the one with the most skills." List them with `jelou project list` and let the person pick.
4. **Never surface a real customer's phone number.** Not in the report, not in chat, not in a log line you print. When you need a live test, use the `jelou test` synthetic tester (see Phase 8) — never a real conversation or a real number pulled from the company's data. This is also a standing org rule: no PII in anything you output.
5. **Every write goes through `jelou push`, never a manual API call, and always resolves drift before pushing.** If `jelou push` reports `LOCKFILE_DRIFT`, follow the `jelou incoming diff` → `accept-local`/`accept-server` flow (Phase 12) — never `--force` without reading the diff first, and never assume you're the only one editing this company's draft.
6. **Check for an existing guard (or a broken hand-rolled attempt) before creating a new one.** Phase 7 is mandatory, not optional, even when you're in a hurry.
7. **AI_TASK / AI_LOGIC prompts: always detect, never silently rewrite.** Grepping their `instructions`/`systemPrompt` text for `$user.id`/`$user.phone` costs nothing extra and has caught the single biggest finding in this project's history (Company C — see `references/patterns.md`). But never edit prompt text as part of the automatic Apply phase; a prompt rewrite always goes into the report as a manual-review item the person acts on themselves, unless they explicitly ask you to also draft the prompt edits.
8. **Never guess whether a `user.id` reference is safe to touch.** Classify every hit as `migrate` / `do-not-touch` / `ambiguous` (Phase 5). An ambiguous hit is reported, never auto-migrated.
9. **A tool's own backing workflow is never edited or re-versioned as part of the normal Apply (Phase 12).** If Phase 6.5 finds `$user.id` misused *inside* a tool's own implementation (not just in how a workflow calls it), that requires its own separate, explicit authorization — asked only after Phase 12 finishes, never bundled into the Phase 11 decision (Phase 12.5). Publishing a new tool version can affect every caller of that tool, possibly outside this project.
10. **Report the true count of tools reviewed.** That means every distinct tool referenced from a standalone `TOOL` node *and* every external tool attached to an `AI_TASK`'s `configuration.connections.tools[]` — not just standalone-node instances. Undercounting here was a confirmed miss (Company E: reported 12, project actually referenced 20+).

## Phase 0 — Parse the request

The invocation looks like: *"usa la skill jelou-phone-migration para la company `<company hint>` del proyecto `<project hint>`"* (or some reordering/rephrasing of the same two facts). Pull out:
- **company hint**: a name or a CLI profile if one is given directly.
- **project hint**: a name, if given — otherwise resolve it in Phase 1.

If the phrasing is ambiguous about which is company vs. project, ask once rather than guessing.

## Phase 1 — Resolve identity and confirm

1. If the company hint looks like an existing CLI profile (pattern seen so far: `daniel<companyslug>`), try it directly: `jelou --profile <p> whoami --agent`. Otherwise, run `jelou whoami --agent` to get the full profile list and try each candidate profile's `whoami` until `company_name` matches the hint case-insensitively. Don't silently pick a near-match — if two profiles could both be it, ask.
2. `jelou --profile <p> project list --agent`. If there's more than one project, match the project hint against `name`/`description`; if still ambiguous or no hint was given, list them and ask.
3. **Stop and confirm with the person**: state the resolved company name, profile, and project name/id, and wait for an explicit go-ahead before touching disk or network beyond read-only `list`/`whoami` calls.

## Phase 2 — Bootstrap a local workspace

- Create a dedicated directory (convention: `<project-slug>-phone-audit`, sibling to wherever you're running from — check it's empty or doesn't exist yet, don't reuse a directory that has unrelated content).
- `jelou --profile <p> link <project-id> --agent`, then `jelou --profile <p> pull --agent`. Read `dirtyConflicts` in the result — if non-empty, stop and surface it; don't proceed over unresolved conflicts.

## Phase 3 — Architecture audit (jelou-graph first, always)

Run `jelou graph summary --agent`. From `architecture.entryCandidates` and the full `jelou workflow list --project-id <id> --agent`, determine:
- Which workflow(s) have `default: true` (there should be exactly one).
- Whether any other workflow has a company-built "router" pattern (an `AI_TASK` classifying intent feeding a `CONDITIONAL`/`CODE` that dispatches via `SKILL` nodes — seen in Company C's "Router principal" and Company D's "2. Router IA principal"). This is **not** the same as Jelou's native AI router — it's an internal re-dispatch hub, note it as such but don't treat it as a second entry point on its own.
- **Which workflows have nothing connected to their `START` node.** For every workflow pulled in Phase 2, check its `START` node's outgoing edges directly in the JSON. A `START` with **zero** outgoing edges means the workflow is a stub — an abandoned draft, an unfinished duplicate, a leftover test — that cannot run for any real user, since nothing executes after `START`. **Exclude every such workflow from Phase 5 onward** (text audit, tool audit, guard placement) — a `user.id` reference sitting in dead code is not a real risk and would only pad the findings with noise. **Never exclude silently**: keep the list (name + id) and put it in the report's Architecture section (Phase 10) so the person can confirm none of them were actually meant to be live. If Phase 4's `jelou workflow evaluate` ever comes back matching one of these as the eligible workflow for a real first message, that's a contradiction — stop and re-verify rather than trusting the exclusion blindly.

Remember `jelou-graph` only models structural relations (`CALLS`/`FLOWS_TO`/`CONTAINS`/`ROUTES_TO`/`READS`/`WRITES`) — it will not find text mentions of `user.id`/`user.phone` inside prompts or config strings. That's Phase 5, done separately, always.

## Phase 4 — Provider + real routing-model check (empirical, not assumed)

1. `jelou channels list --include-sandbox --agent`, filter to this `projectId`. Read `provider`. If there is no WhatsApp channel at all, or more than one with different providers, stop and flag it in the report as a blocker/open question rather than guessing.
2. Classify the provider: `whatsapp_cloud` / `gupshup_capi` / `aldeamo` → native `contact_info_request` ("Solicitar contacto") button is usable. Anything else (plain `gupshup`, or unrecognized) → button unavailable, the guard must fall back to a manual `INPUT` node (see `references/patterns.md`).
3. **Never assume the routing model from workflow count.** Run `jelou workflow evaluate --message "<generic greeting>" --expected <defaultSkillId> --agent`, then run it again with 2-3 plausible first-messages targeting other non-default workflows that showed up with `user.id` hits in Phase 5. Read `routingPath`:
   - `fallback` or `single_eligible` (even when testing the default skill on purpose) → **no real AI routing**. Bucket: single default entry, or legacy menu if many workflows are only reachable via internal `SKILL` calls from one root.
   - `embedding_direct` or `llm_verdict` with `match: true` → **real AI routing is active**. Every workflow that (a) has a `user.id` finding from Phase 5 and (b) matches a plausible first message needs its own guard, not just the default.
4. Record which bucket this company is in — it drives Phase 9 entirely.

## Phase 5 — Text audit for `user.id` / `user.phone`

Do NOT rely on `jelou graph` for this (see Phase 3 note). Pull every workflow JSON already on disk from Phase 2, **excluding any workflow Phase 3 flagged as having nothing connected to its `START`**, and search the rest directly, across **every** node type including `AI_TASK`/`AI_LOGIC` `instructions`/`systemPrompt` (Rule 7) — never restrict the search to structural nodes only, even if a person tells you to skip *designing fixes* for AI nodes; detection is always in scope.

Patterns to search for (see `references/patterns.md` for the exact regex list and why each one matters):
- `{{$user.id}}` / `$user.id` in template fields (CONDITIONAL terms, DATUM queries/rows, HTTP bodies/URLs, MEMORY variables, CHANNEL_MESSAGE text used as a payload).
- `$user.get("id")` and `$user.get("phone")` inside CODE node `content` — the second one is always a live bug (see Rule below and `references/patterns.md`).
- The `$user.get("phone") || $user.get("id")` (or equivalent try/catch) anti-pattern — flag explicitly, it never falls through to `id`.
- `{{$user.phone}}` mentions, especially inside AI_TASK prompts — count them; a high count is a strong signal to prefer the native button design in Phase 9 even when it costs more to set up, because it avoids touching every prompt.
- Every `AI_TASK`'s `configuration.settings.contextBlocks.userInfo` — if it's `true`, or an object with `enabled: true` and no `fields` override (or `fields` including `"id"`): this structurally injects the user's `id` into that model's context on every turn. **This is not migratable the normal way** — the engine strips `phone` from this block unconditionally ("legalId and phone are stripped as unsafe" per `jelou workflow node-spec AI_TASK`), so there is no config flag to make it inject `phone` instead. Flag it as its own finding: the model sees a possibly-unreliable `id` with no structural way to give it `phone` here — the only fix is to stop relying on this block for phone and thread the resolved value into `instructions` explicitly (e.g. mention `{{$user.phone}}` or the guard's memory variable in the prompt text itself), which is a prompt edit and therefore governed by Rule 7 (report, don't auto-rewrite).

For every hit, read the **whole node's configuration**, not just the matching substring, and classify:
- **`do-not-touch`**: the value feeds an internal Jelou platform endpoint — the signature is a URL shaped like `api.jelou.ai/v1/bots/{botId}/users/{id}/...` (seen: `/skill/...`, `/storedParams/legacy`). These need a stable, never-empty session identifier, which is exactly what `$user.id` provides and `$user.phone` does not (it can be empty for the very users this project protects). Explain this in the report so nobody "fixes" it later by mistake.
- **`migrate`**: the value is real business data — a phone/celular/telefono field read by a CRM, a shipping carrier, a Datum table, a payment/registration record, or passed to an external tool. Read the *full* payload of any `TOOL`/`HTTP` node this touches (Phase 6) before finalizing priority.
- **`ambiguous`**: you can't tell from the node alone (generic field names like `userId` sent to an unfamiliar external API, or a value whose downstream consumer isn't visible from this project). Report it as needing a human decision — never guess.

## Phase 6 — Tool/API payload depth pass

For every `migrate` or `ambiguous` hit that lives inside a `TOOL` or `HTTP` node, read the entire `configuration` (URL, method, full body/input, not just the field with `user.id`) and figure out where it actually goes:
- **External, customer-facing service** (a shipping carrier, a CRM record a human will read, a payment processor, a registration the customer would notice if wrong) → **High** priority. Getting this wrong has real-world consequences (Company D: a wrong phone on a physical shipping guide).
- **Internal bookkeeping the company's own team reads** (an error-alert email, an internal log/activity table) → **Low** priority — still worth fixing, but not urgent.
- Anything in between → **Medium**, and say why.

Don't stop at the first field that matches — the same tool node can carry the identifier in more than one place (a login call and a diagnostic email calling the same value for different reasons look identical in a shallow grep).

## Phase 6.5 — Tool inventory and internals audit

Phase 5/6 above only sees what the *calling workflow* passes into a tool — not what the tool does with it internally, and a tool's own backing workflow is never part of `jelou pull` (it's a separate graph object, `availability: referenced` not `local` in `jelou graph summary`). This phase closes that gap. Skipping it was a confirmed miss (Company E: a tool's own internal DATUM lookup by `userid` almost went unreported).

1. **Build the true tool inventory** — every distinct external tool the project actually calls, from two sources:
   - Standalone `TOOL` nodes: `configuration.toolData.toolId` (or `configuration.toolId`).
   - Every `AI_TASK`/`AI_LOGIC` node's `configuration.connections.tools[]` entries that carry a `toolId` (these are the *external marketplace* tools, distinguishable from Jelou's built-in native tools — `search_products`, `transfer_to_agent`, `memory_manager`, `datum_records`, etc. — which set `external: false` and have no backing workflow to pull; skip the native ones, they're platform code, not company-authored). Also check `configuration.connections.mcp[]` for any entry with a `url` pointing at a company-run MCP server (rare, but if present, note it as out of scope for this audit — a live MCP server isn't a pullable workflow).
   - Report the **distinct count** from both sources combined as "tools reviewed" — never just the standalone-node count (Rule 10).
2. For each distinct `toolId`: `jelou tool show <toolId> --agent` to get its `workflow_id`/slug, then `jelou tool pull <slug> --agent` to fetch its backing workflow into `tools/<slug>.json`.
3. Run the **same Phase 5 pattern search and Phase 6 payload-depth read** against that pulled file — a tool's own workflow is a workflow, audit it identically.
4. Any hit found *inside* a tool's own implementation gets a distinct tag: **`tool-internal`** — never `migrate`/`do-not-touch`/`ambiguous` (those describe a calling workflow's own field). A `tool-internal` finding cannot be fixed by editing the calling company's workflows; it requires editing the tool's source and publishing a new version (Rule 9, Phase 12.5).
5. For every `tool-internal` finding, enumerate **every caller** of that `toolId` across the whole project (every `TOOL` node and every `AI_TASK` `connections.tools[]` entry referencing it, in every workflow, not just the ones already in scope) — fixing a shared tool affects all of them at once, and the report needs to show that blast radius before anyone authorizes touching it.

## Phase 7 — Existing-guard / broken-attempt detection

Before designing anything, check whether this company already has:
- A workflow titled exactly `Guardia Teléfono` (the standard name, Phase 9) — check this first, it's the fast path. If absent, fall back to a looser match: any workflow whose name/purpose looks like a phone-resolution guard (title containing "telefono"/"contacto"/"phone", a `CONDITIONAL` on `$user.phone` feeding a `contact_info_request` or `INPUT`). Either way, if found, note its `skillId` and reuse/extend it in Phase 9 instead of building a duplicate.
- A broken hand-rolled attempt (the `$user.get("phone") || $user.get("id")` pattern, or any other defensive-looking code that would in fact always fail per `references/patterns.md`). Report it as a live bug, distinct from "nothing done yet."

## Phase 8 — Runtime verification (automatic, always cleaned up)

Do this for real evidence, not assumption — but every step here writes to the company's draft temporarily, so treat it as scaffolding that must always come back out.

1. Pick the cheapest reachable workflow to host a temporary probe (prefer a tiny one, like a "salir"/goodbye workflow, over the default entry, to minimize blast radius).
2. Insert a `MEMORY` node right after its `START` templating `{{$user.phone}}` into a scratch variable, so you can confirm — for *this* company's specific provider and synthetic tester — that it resolves safely to `""` with no throw. **Do not write `$user.get("phone")` into a CODE node as part of this check** — that it throws is already a confirmed, documented platform fact (`references/patterns.md`); re-proving it per company adds a needless CODE node and risk for zero new information. The only thing worth verifying live, per company, is the template-form behavior and (if Phase 7 found a hand-rolled CODE pattern already using `$user.get("phone")`) confirming that *specific* existing node actually fails the way `references/patterns.md` predicts, by reading its trace from a normal test turn rather than writing a new probe for it.
3. `jelou workflow validate` the file, fix any ID-format issues (nanoid errors — regenerate with a proper 21-char alphanumeric+`_-` id), then `jelou push`.
4. `jelou test session start`, `jelou test send text`, then `jelou test trace --include-raw` and read the diagnostic node's `finalState.logs`.
5. **Always revert**, regardless of what the test showed or whether any step above failed partway: remove the temp node/edges, restore the original edge, `jelou workflow validate` again, `jelou push` again. If a push after revert fails on an unrelated pre-existing gate (short description, ambiguous model — very common, see `references/patterns.md`), fix only what's needed to get the file back to clean and pushed; don't fix unrelated things you didn't come here for. If you cannot get the file back to a clean pushed state, say so explicitly and clearly rather than leaving it unmentioned.

## Phase 9 — Design the guard placement plan

Using the Phase 4 bucket:

- **No real AI routing** (single default, or legacy menu behind one root): wire the guard exactly once, at the true entry — a `SKILL` node calling the guard workflow, placed right after that workflow's `START`, before anything else runs.
- **Real AI routing**: wire the guard at every workflow that both (a) has a `migrate`-classified `user.id` finding and (b) was confirmed reachable as a first message in Phase 4. Don't add it to workflows with no phone-dependent logic just for symmetry.

### The guard is always its own workflow — never inline it into the caller

**Never create the guard's `START`/`CONDITIONAL`/`CHANNEL_MESSAGE` (or `INPUT`)/`END` nodes directly inside the entry or target workflow.** The guard is always a separate, standalone, `SKILL`-callable workflow with its own `START` and its own `END` declaring the single `resuelto` output (the canonical shape below describes *that* workflow's internals). Every workflow that needs the guard gets exactly one `SKILL` node, wired right after its own `START`, calling the guard workflow — the caller's own `START` is never touched beyond adding that one `SKILL` node and routing its `resuelto` output onward. Confirmed live mistake (Company G): the guard's nodes were built directly inside the `inicio` workflow instead of as their own workflow. This pollutes the entry workflow's graph, breaks reuse across the multiple entry points a real-AI-routing bucket needs (each one would otherwise need its own copy instead of one shared `SKILL` call), and means Phase 7's existing-guard detection in a future run can't find/extend it cleanly. Always create the guard as its own workflow first (`jelou workflow create`), get it pushed and validated on its own, then wire callers to it via `SKILL` nodes.

### The canonical shape — always this, never more

The guard is always exactly this shape, regardless of company. **Never add a second declared output, never add a branch that escalates to a human/advisor/PMA workflow, and never add any node beyond what's listed below** — even if the company already has an advisor-handoff workflow sitting right there and it would be "easy" to wire a failure path into it. One run (Company F) drifted into adding a PMA-escalation output that was never asked for; treat that as the specific mistake to not repeat. If a real gap ever seems to call for more than this shape, stop and ask the person instead of improvising it into the design.

**Standard workflow name: `Guardia Teléfono`.** When minting a new guard (no match found in Phase 7), always create the workflow with this exact title — never a company-specific variant. This makes Phase 7's existing-guard detection in every future run on this project a direct name match instead of a fuzzy title guess. If Phase 7 finds an existing guard under a different title (a pre-`Company G` migration, or a company's own hand-rolled attempt), reuse/extend its `skillId` as before (don't rename a live workflow just to match), but note the title mismatch in the report.

Exactly **one** declared output on the guard skill (name it `resuelto`, display "Telefono Resuelto" or equivalent) — never a second `no_resuelto`/`failed` output.

**Never wire an edge from a node back to itself.** It passes local `jelou workflow validate` but the server rejects the actual edge-creation call with `Insufficient scope / FORBIDDEN` (confirmed live against Company E) — a self-loop is not a supported edge shape on this platform, full stop, regardless of node type. Any "retry" edge below must target a *different* node than its own source — the pattern is to bounce back to the `CONDITIONAL` (node 2), never to the node that just failed.

**Native button available:**
1. `START`
2. `CONDITIONAL` "¿Teléfono vacío?" — one term, `{{$user.phone}}` `is_empty`.
   - condition (empty) → node 3.
   - `else` → node 4 directly (already resolved, nothing to ask).
3. `CHANNEL_MESSAGE` "Solicitar contacto" — `contact_info_request`, `settings.isBlockingEnabled: true`, a short text such as "Para continuar necesitamos confirmar tu número de contacto. Usa el botón para compartirlo."
   - `default` (contact shared) → node 4.
   - `expire` → **omit this edge.** A blocking `contact_info_request`'s validator rule technically wants `expire` wired, but the only two ways to satisfy it (loop to itself, or a second declared output) are both ruled out (self-loop rejected server-side; second output forbidden by this shape). Leaving it unwired matches the real, already-live Company A design and is the accepted trade-off: `jelou workflow validate` will report one `edge_error_missing_expire_branch_required_when_contact_info_request_is_blocking`, note it in the report as a known, intentional gap, and push anyway — the server has accepted this shape in practice. If a future platform change makes a non-self-loop `expire` edge possible, prefer wiring it back to node 2 instead of leaving it unwired.
4. `END` "Éxito" selecting the single `resuelto` output.

**Native button unavailable (INPUT fallback):** same shape, node 3 becomes `INPUT` ("¿Cuál es tu número de teléfono de contacto?") wired `success` → a `CODE` node that validates digits (8–15) and `$memory.set("telefono_resuelto", ...)` on success, going to node 4; on invalid digits, loop back to **node 2** (the `CONDITIONAL`), not to the `INPUT` node itself. The `INPUT`'s own `expire`/`exit` also target node 2, not itself. Every downstream `migrate` hit then points at `{{$memory.telefono_resuelto}}` instead of `{{$user.phone}}`.

If an existing guard was found in Phase 7, extend/reuse its `skillId` instead of minting a new one — and if that existing one has drifted from this shape (an extra output, a stray branch), say so in the report as a warning rather than silently leaving it or silently "fixing" it without the person's sign-off.

## Phase 10 — Generate the report

Build a single self-contained HTML artifact (load the `artifact-design` skill before writing it if it isn't already loaded this session) using `references/report-template.html` as the starting structure — fill in real findings, don't ship the placeholder copy. Sections, top to bottom:

1. **Header**: company name, project name, as-of date, and status badges (provider, routing-model bucket).
2. **Summary tiles**: counts — files audited, nodes to migrate, do-not-touch, ambiguous, tools/APIs reviewed.
3. **Architecture**: one short paragraph + the routing-model evidence (the actual `jelou workflow evaluate` results, not just the conclusion), plus the list of workflows Phase 3 excluded for having nothing connected to their `START` (name + id, or "none" if all were live).
4. **Findings table**: one row per hit, columns for workflow, node, classification (color-coded: migrate/do-not-touch/ambiguous), and a one-line reason.
5. **Tools & APIs**: the Phase 6 priority list (High/Medium/Low) with the real destination named. State the true tool count from Phase 6.5 (e.g. "20 tools revisadas: 12 en nodos TOOL, 8 conectadas a agentes IA"), not just standalone-node instances.
5b. **Tools con lógica interna afectada** (only when Phase 6.5 found `tool-internal` hits): one entry per affected tool — its name/id, its own workflow, what's wrong inside it, and the full list of callers across the project (file + node, both `TOOL` nodes and `AI_TASK` connections). Make clear in the copy that fixing this is a separate step gated on its own authorization (Phase 12.5), not part of the main plan.
6. **Existing state**: what Phase 7 found, if anything.
7. **Runtime verification**: what Phase 8 actually showed (quote the real error/result, not a paraphrase).
8. **Plan**: exactly what Phase 9 would create/change, file by file.
9. **Warnings**: anything unresolved — ambiguous hits, missing channel, provider limitations, anything Phase 8 couldn't fully verify.
10. **Decision**: a plain checklist of the options (see Phase 11) — this section is content the person reads, the actual choice happens in chat.

Never put a real phone number anywhere in this artifact (Rule 4) — use the synthetic test value or a placeholder like `593900000000` in examples.

## Phase 11 — Decision gate

After publishing the artifact and sending the person its link, ask explicitly (in chat, e.g. via a clarifying question) which they want:
- Apply the plan as written.
- Apply it with specific adjustments (get the adjustments first, in their own words, before touching anything).
- Don't apply anything — stop here.

Do not proceed to Phase 12 without one of these being explicit and current (a stale "yes" from earlier in a long conversation about something else doesn't count).

## Phase 12 — Apply (only after explicit approval)

Execute Phase 9's plan file by file: create/extend the guard **as its own standalone workflow** (never as nodes inserted into the entry/target workflow itself — see Phase 9), wire it at the determined entry points via a `SKILL` node placed right after each target workflow's own `START`, migrate every `migrate`-classified hit (never `ambiguous` ones — those stay as reported warnings unless the person addressed them explicitly in their approval). For a `migrate`-classified hit inside a CODE node calling `$user.get("id")`, replace it directly with `$user.get("phone")` — this works as a straight literal swap, with no `MEMORY`/`$memory.get()` workaround and no dedicated runtime probe needed (see `references/patterns.md`, Company G). For every `jelou push`:
- Resolve `LOCKFILE_DRIFT` via `jelou incoming diff` first, then `accept-local` only after confirming the diff shows your own intended changes and nothing you'd be clobbering.
- Fix pre-existing push-blocking gates you encounter along the way (short `workflowDescription`, ambiguous/unqualified AI model names) using the same conventions as the rest of the file you're already touching — but don't go hunting for unrelated ones in files you have no other reason to touch.
- When a pre-existing gate needs a **destructive** fix to clear (deleting a node, not just editing a description or a model string), never do it silently even if you're confident it's safe (e.g. confirmed unreachable via `jelou graph`/no incoming edges). Stop and ask first, the way it played out in Company E (`TwLJK2I1B9Fq0x2JtAFni`, an orphaned WhatsApp Flow node blocking `inicio`'s push) — that back-and-forth is the intended behavior, not friction to engineer around.
- A self-referencing edge (`sourceId === targetId`) always gets rejected by the server with `Insufficient scope / FORBIDDEN`, even when `jelou workflow validate` accepted it locally — don't attempt one (see Phase 9's canonical shape and `references/patterns.md`), and if you ever hit this exact error elsewhere, the fix is the same: remove the self-loop, either leave the edge unwired if that's tolerable or retarget it at a different upstream node.
- After every file is clean and pushed, run `jelou status --agent` and confirm `modified: 0` before declaring done.
- **Do not run `jelou project publish`** (Rule 1). Tell the person everything is in draft and ready for them to test and publish when they're ready.

## Phase 12.5 — Tool-internals fix (separate authorization, only if Phase 6.5 found any)

Only reached when Phase 6.5 tagged at least one `tool-internal` finding, and only after Phase 12 (or the person's explicit decision not to apply the workflow-level plan) is settled. This is a **second, independent** yes/no — approving Phase 11's plan never implies approving this one.

1. Before asking, run `jelou tool versions <slug> --agent` for each affected tool: if it has callers or published versions that don't belong to this project/company, say so explicitly — bumping the version could affect consumers outside the scope of this migration.
2. Ask explicitly what this step would do, in plain terms: *"¿Quieres que también corrija la(s) tool(s) [nombres] que usan `$user.id` internamente? Esto: (1) edita la implementación de la tool, (2) publica una versión nueva (`jelou tool publish --version X.Y.Z`), y (3) actualiza todas las llamadas — nodos TOOL y conexiones de agentes IA, en todos los workflows del proyecto — a esa nueva versión (`jelou tool bump`)."* Include the caller list from Phase 6.5 point 5 so the person can see the full blast radius before deciding.
3. Only on explicit yes: `jelou tool pull <slug>` (if not already local from Phase 6.5), fix the same way Phase 12 fixes a workflow (migrate/do-not-touch/ambiguous classification still applies inside the tool's own graph), `jelou tool push`, then `jelou tool publish --version <next>` — pick a patch bump unless the person specifies otherwise — then `jelou tool bump <slug> --to <version>` for every caller found. Confirm with `jelou tool versions <slug>` that callers now point at the new version.
4. This step still never publishes the *project* (Rule 1) — a tool version publish is a different, narrower action than `jelou project publish` and is what this step is explicitly authorized to do; the project's own draft/publish state is untouched by it.

## Phase 13 — Close out

One short message: link to the report (if not already sent), a one-line summary of what was applied vs. left as warnings, and confirmation that nothing was published. If memory files for this project exist (see a Jelou CLI project's own memory conventions, if any), consider noting anything genuinely new learned about the platform itself (not company-specific facts) — but only if it's a real generalizable finding, not company trivia.
