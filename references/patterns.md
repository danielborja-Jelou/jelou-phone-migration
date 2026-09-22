# Known patterns — evidence from real migrations

Every rule here was confirmed empirically (via `jelou test` traces or `jelou workflow validate`), not inferred. Company names in parentheses are where it was first confirmed; treat the pattern as platform-wide unless noted otherwise.

## This is a WhatsApp-only concern — scope every audit to `.whatsapp.json`

`$user.phone`, `contact_info_request`, and the whole BSUID-unresolved-identity problem this skill exists to fix are specific to how WhatsApp identifies a contact. A Web session has no phone number at all (its `$user.id` is a browser/session identifier); a Facebook conversation's `$user.id` is a page-scoped PSID; Instagram's is an IGSID — neither channel has a `phone` concept for `$user` to resolve, and `contact_info_request` isn't even a valid message type off WhatsApp (per `jelou workflow node-spec CHANNEL_MESSAGE`'s channel-availability matrix: WEB/FACEBOOK/INSTAGRAM/TWITTER each have their own distinct palette of message types, and none of them include it). Auditing a non-WhatsApp workflow for `user.id`-as-phone misuse is a category error — there's no "correct" value to migrate to.

**Mechanical rule:** every pulled workflow file is named `<slug>.<channel>.json`. Treat anything other than `.whatsapp.json` as out of scope from Phase 3 onward, and a company with no `.whatsapp.json` files at all as one this skill has nothing to offer — say so and stop rather than producing a report with zero real findings that reads as "all clear." A company with WhatsApp plus other channels is the common case, not an edge case: scope down to WhatsApp, don't treat the other channels as an error condition.

## Search patterns for Phase 5

Run these across every pulled workflow JSON file, on the raw text (not just parsed node configs), so you also catch matches inside `AI_TASK`/`AI_LOGIC` `instructions`/`systemPrompt` strings:

```
user\.id
user\.get\(\\?"id\\?"\)
user\.get\(\\?"phone\\?"\)
\$user\.phone
```

Note the JSON-escaping: a pulled workflow file has literal `\"` inside string values, so `$user.get("phone")` appears in the raw bytes as `$user.get(\"phone\")`. Grep for both the escaped and unescaped forms, or normalize by loading the JSON and re-serializing before searching node-by-node.

## `$user.get("phone")` in CODE nodes works on real WhatsApp — the synthetic tester just can't confirm it

**Root cause found (2026-09-21/22, confirmed across multiple real companies over real WhatsApp):** `$user.get("phone")` works fine inside a `CODE` node — a direct swap of an existing `$user.get("id")` call for `$user.get("phone")` is safe and is now the standing migration move. There is no meaningful difference between "replacing an existing call" and "writing it from scratch" — the sandbox executes whatever text is in `content` regardless of how it got there; a distinction between those two never made sense and earlier phrasing here was wrong to imply one.

**Why the earlier evidence looked so solid and was still wrong:** every prior confirmation of `$user.get("phone")` throwing `Error: Key phone not found.` (Company A, Company B, and the Phase 8 probes run since) was captured through `jelou test` — the **synthetic tester**, never a real WhatsApp conversation. The synthetic tester's `$user` object does not carry a `phone` key **at all**, because a synthetic test user is not a real, phone-resolved WhatsApp contact — it's not that the value is empty, the key is structurally absent for that harness. So `$user.get("phone")` throwing there is a property of the *test tool*, not of the platform: it will throw on every synthetic-tester probe, for every company, forever, regardless of whether real customers would ever hit that line. Treating that as a platform limitation was a reasonable-looking but incorrect inference from a tool that wasn't built to represent this particular field.

**Standing rule going forward: never use `jelou test` (the synthetic tester) to verify anything about `$user.get("phone")` or the presence of the `phone` key on `$user` in a CODE node.** A "throws" result from that specific probe is expected and uninformative — it proves nothing about real customers. If Phase 8's automatic verification needs evidence for a phone-migration change, keep it scoped to what the synthetic tester *can* represent correctly: the **template** form `{{$user.phone}}` (a `MEMORY` node with `"variables": {"some_key": "{{$user.phone}}"}`) does safely resolve to `""` on the synthetic tester with no throw, and that read *is* representative — a template reference behaves the same for a real unresolved contact and for the synthetic one, since both simply lack a phone value; it's only the CODE-sandbox `.get("phone")` accessor whose behavior diverges between the two. When a company's real behavior needs confirming beyond that, it takes an actual WhatsApp conversation (live channel), not `jelou test` — flag that distinction in the report rather than asserting confidence the tool can't back up.

To get the resolved phone into a CODE node when there's no existing `$user.get("id")` call to swap (e.g. building new logic, not migrating an old line): just call `$user.get("phone")` directly — it works. The old `MEMORY` + `$memory.get()` workaround documented in earlier versions of this file is no longer necessary for this specific case; keep it in mind only as a generic pattern for bridging a template-only value into CODE, not as a required workaround for `phone`.

## `$output.set(...)` is invalid in a plain CODE node

`jelou workflow validate` rejects it outright with code `config_code`: *"CODE nodes have no declared outputs, so it throws 'Key X not found' at runtime."* Its own suggested fix is exactly the `$memory.set` pattern above. `$output.set` only makes sense inside a workflow that declares outputs consumed by an `END` node (a TOOL's own backing workflow, or a SKILL-callable workflow with `jelou workflow output add`).

## The `$user.get("phone") || $user.get("id")` anti-pattern always fails to fall through

Found live in 4 CODE nodes in Company C (all named things like "Gate: teléfono"), always inside a try/catch:

```js
try { telefono = String($user.get("phone") || $user.get("id") || ""); } catch (e) { telefono = ""; }
```

In JS, if the first operand of `||` throws, the exception propagates immediately — `$user.get("id")` is never evaluated, so if `$user.get("phone")` throws, `telefono` always ends up `""` (or whatever the catch branch defaults to), never the id.

**This finding needs re-verification before being treated as a confirmed bug, in light of the correction above.** It was originally diagnosed by reasoning from the (incorrect) belief that `$user.get("phone")` always throws for every conversation — that belief came from synthetic-tester evidence only (see above), and it's now established that `$user.get("phone")` works on real WhatsApp. If it also doesn't throw for a real BSUID-unresolved contact on real WhatsApp (returning a falsy value instead), this exact pattern would actually fall through to `$user.get("id")` as originally intended, and would **not** be a bug at all. Don't re-assert this as a confirmed live bug purely from reading the code anymore — it needs an actual real-WhatsApp check (or the company's own runtime logs) before being reported as broken, not just the JS-semantics argument alone. If you find this shape again, report it as "needs verification" rather than "confirmed bug" unless you have real-conversation evidence either way.

## `contact_info_request` ("Solicitar contacto") is provider-gated

Per `jelou workflow node-spec CHANNEL_MESSAGE`: the `contact_info_request` message type only works when the channel's provider is `whatsapp_cloud`, `gupshup_capi`, or `aldeamo`. Plain `gupshup` (classic, non-CAPI) does not support it — `jelou workflow validate --provider gupshup` rejects it outright. Always check with `jelou channels show <id>` / `jelou channels list --include-sandbox` before designing the guard.

When it *is* available and the user shares their contact once, `$user.phone` gets resolved at the platform level for the rest of that conversation — every downstream `{{$user.phone}}` reference (including ones already inside `AI_TASK` prompts) starts working with zero further changes. This is why, when a company already has many `AI_TASK` prompts referencing `{{$user.phone}}` (found: Company C, 10 prompts across 6 workflows, used for CRM identity matching and as parameters into Datum tool calls), the native button is worth the extra setup — the alternative means editing every one of those prompts.

When it's *not* available, there is no way to write into `$user.phone` from anywhere (CODE sandbox globals are `$memory, $context, $input, $output, $env, $secret, $user (read-only), $bot, $company, $message, $webhook, $utils` — no setter). A manually-typed answer can only live in `$memory`, and every downstream node must be pointed at that memory key instead of `$user.phone`.

## The guard's `expire` branch: no second output, and no self-loop either

A blocking `contact_info_request` needs `default` + `expire` wired (`jelou workflow validate` blocks otherwise), but a naive design where `default` and `expire` both eventually reach the same downstream node can double-fire it — production evidence in Company A showed `expire` dispatching milliseconds after a valid `default` resumed, running the rest of the flow twice. Company A's own fix was to delete the `expire` edge entirely, which resolves the double-fire but reintroduces the validator error (missing required `expire` branch) — accepted there because nobody enforced it before publish.

**Confirmed 2026-09-20, Company E:** the natural-looking alternative — wire `expire` back to the `contact_info_request` node itself, a self-loop — looks valid to `jelou workflow validate` (offline) but the live server rejects the edge-creation call with `Insufficient scope / FORBIDDEN`. Self-referencing edges (a node's own id as both `sourceId` and `targetId`) are not a supported shape on this platform, at least not through this API path — don't design around them, for this guard or anything else that might seem to want a "retry itself" edge.

**Current standing pattern:** leave `expire` unwired on the `contact_info_request` node (same outcome as Company A's manual fix) — `jelou workflow validate` reports one `edge_error_missing_expire_branch_required_when_contact_info_request_is_blocking`, note it as a known accepted gap in the report, and push anyway; it has gone through in practice. For the `INPUT`-fallback variant's retry logic (invalid digits, or its own `expire`/`exit`), route back to the upstream `CONDITIONAL` node instead of the `INPUT` node itself — a two-node cycle is fine, only a literal self-loop is rejected. See Phase 9 in `SKILL.md` for the canonical shape this produces.

## The guard must be its own workflow — never nodes inlined into the caller (Company G)

Confirmed live mistake, Company G: when applying the fix, the guard's `START`/`CONDITIONAL`/`CHANNEL_MESSAGE`/`END` nodes were created directly inside the `inicio` (entry) workflow instead of as their own separate workflow. This is wrong regardless of routing bucket:
- It pollutes the entry workflow's own graph with nodes that belong to a reusable sub-flow.
- In the real-AI-routing bucket, every matching workflow needs the same guard — inlining it means duplicating the same nodes into every workflow instead of adding one `SKILL` node per workflow that all call the same guard.
- A future Phase 7 existing-guard detection pass looks for a distinct workflow with guard-shaped nodes; nodes buried inside an unrelated entry workflow are much harder to recognize and reuse.

**Standing rule:** the guard is always authored as its own standalone, `SKILL`-callable workflow (its own `START`, ending in an `END` that selects the single `resuelto` output). Every caller — the true entry in the no-real-routing bucket, or each matching workflow in the real-AI-routing bucket — gets exactly one `SKILL` node wired right after its own `START` that calls the guard workflow, never the guard's raw nodes copy-pasted or hand-built in place.

## Being its own workflow isn't enough — the guard also needs a real declared output

Per `jelou workflow node-spec SKILL`: a `SKILL` node calling another workflow has two modes decided purely by whether the caller wires any `output:<id>:*` edge. With none, it's "legacy handoff" — the child starts and the caller keeps going immediately on `default`, without waiting for anything. Only wiring at least one `output:<id>:*` edge makes the caller park and resume when the child's `END` node selects that output. So a guard workflow that has its own `START`/`CONDITIONAL`/`contact_info_request`/`END` nodes but **never declared an output** (`jelou workflow skill <skillId> --agent` → `"outputs": []`) is only usable in the useless mode — every caller wired to it would fire it and immediately continue without the phone ever being resolved, silently defeating the whole guard.

`jelou workflow skill <skillId> --agent` is the check: it returns `outputs[]` as `[{ id, name, display_name, type, position }]` (`type` is `SUCCESS` / `FAILED` / `DEFAULT` — the caller's branch token is `output:<id>:<type lowercased>`). Real example, Company A's original guard (skillId 50330, before the single-output shape was standardized):

```json
"outputs": [
  {"id": "_Lnf9nS-00-DMbtt1jkYs", "name": "no_resuelto", "display_name": "Telefono no resuelto", "type": "FAILED"},
  {"id": "1M5osuMVsbCFWpbeI0BmY", "name": "resuelto", "display_name": "Telefono resuelto", "type": "SUCCESS"}
]
```

**Only the `SUCCESS` output matters — this skill never designs around, wires against, or maintains any other output a guard happens to have.** `contact_info_request` has exactly one real way out once `expire` is left unwired (see above): `default`. One way out means one output, full stop — Company A's leftover `no_resuelto`/`FAILED` above is unused legacy metadata to leave alone, not something to reconcile or wire a caller's second edge to.

Two consequences: (1) if Phase 7 finds an existing guard with no `SUCCESS` entry in `outputs[]` (empty, or only non-`SUCCESS` ones), adding it (`jelou workflow output add <skillId> --name resuelto --type SUCCESS ...`) plus an `END` node that selects it is a required fix before the guard can be safely reused, not optional polish; (2) when wiring any caller — new or existing guard — always read the real `SUCCESS` output's `id` from this command and use `output:<that id>:success` verbatim. Hardcoding the canonical shape's assumed id against a guard that actually uses a different one produces a `SKILL` node that looks correctly wired in the JSON but never resumes — no validator error, no runtime error, it just parks forever.

## Distinguishing "platform-identity field" from "business phone field"

A `$user.id` reference feeding a URL/param shaped like `api.jelou.ai/v1/bots/{botId}/users/{id}/...` (seen: `/skill/<id>`, `/storedParams/legacy`) is the platform's own session/user key — it must never be empty, which is exactly why it should stay `$user.id` and not become `$user.phone`. `$user.phone` can legitimately be empty for the very BSUID-unresolved users this project protects; swapping there would send a malformed path (`/users//...`) for precisely those users. This is a `do-not-touch`, not a bug.

Contrast with genuine business-phone fields: a `telefono`/`Celular`/`phone`/`userId` (confusingly named, but confirmed by context) field read by a CRM, a shipping carrier, a payment tool, or a Datum table. Read the whole payload of the node, not just the field name, to tell which case you're in — a generic `userId` sent to an unfamiliar external API with no other context is `ambiguous`, not an automatic `migrate`.

## Tool/API blast-radius examples (Company D)

Same literal shape (`"Usuario": "{{$user.id}}"` inside a TOOL node's body) meant wildly different things depending on where it actually went:
- Inside a `[Utility] Dynamic Email` tool whose body was an internal alert to the company's own sysadmin address about a login-service failure → **Low** priority, purely diagnostic.
- Inside a `CHANNEL_MESSAGE`/HTTP payload being built for a shipping carrier's actual guide-creation API (a field like `telefono1_destinat_ne` — the carrier's own naming convention) → **High** priority; a wrong value here means a real physical shipping label with the wrong delivery contact number.
- Inside `DATUM` writes to `Celular`/`userID` fields on Invoice/Order CRM tables → **High** priority, real business records.

Always trace to the real destination before assigning priority — don't assume from the field name alone.

## Routing-model signals from `jelou workflow evaluate`

| `routingPath` in the result | Meaning |
|---|---|
| `fallback` | No eligible workflow matched — router essentially inactive for this message. |
| `single_eligible` | Only one workflow is even a candidate (usually the default) — no real routing happening regardless of workflow count. |
| `embedding_direct` | A real embedding-similarity match picked a specific workflow — AI routing is genuinely active. |
| `llm_verdict` | An LLM step confirmed the match after embeddings — also genuine AI routing. |

Test with the default skill's own greeting-style message too — if even that comes back `single_eligible`/`fallback`, the company has no real AI router access at all, no matter how many workflows exist or how good their descriptions are (confirmed for Company A, Company B, Company C; contrast with Company D where `embedding_direct`/`llm_verdict` showed up immediately).

A company-built "AI router" workflow (an `AI_TASK` classifying intent → `CONDITIONAL`/`CODE` → `SKILL` dispatch, e.g. Company C's "Router principal", Company D's "2. Router IA principal") is **not** the same thing as the platform's native router tested above — it's an internal re-dispatch hub, typically only reachable via `SKILL` calls from workflows already downstream of the one true entry. Don't count it as a second entry point on its own; verify with `jelou graph references <it> --direction in` whether anything besides internal `SKILL` calls points at it.

## Push-time gates you will hit on almost every file, regardless of company

Two validation gates are unrelated to this migration but will block `jelou push` on nearly any file you touch:
- `workflowDescription` under ~30 characters on a non-default, non-hidden workflow → "the AI router's embedding + LLM verdict pipeline needs >=30 chars of intent context." Fix by writing a real, specific, action-verb description of what the workflow does.
- An AI model name that's ambiguous (`gpt-4o` served by multiple providers) or fully unknown (`gpt-4o-azure` isn't a real qualified id) → qualify it (`openai/gpt-4o`, `azure/gpt-4o`) matching the convention already used by sibling `AI_TASK` nodes in the same file when there's a clear one, or ask the person if there's genuinely no signal either way.

Fix only what's blocking the push of a file you already have another reason to touch — don't go looking for these across the whole project.

## A tool's own backing workflow is invisible to a normal audit

`jelou pull` only fetches the project's own workflows. A `TOOL` node's `configuration.toolData.toolId` points at a *separate* graph object (`jelou graph summary` shows it as `availability: referenced`, not `local`) — its own implementation lives in its own workflow, fetched separately with `jelou tool show <toolId>` (to get the `workflow_id`/slug) then `jelou tool pull <slug>`. Confirmed live on Company E: a tool's own internal `DATUM` lookup by `userid` almost went unreported because the audit only read what the *calling* workflow passed into the tool, never the tool's own graph. Any migration audit that stops at the calling workflow is structurally incomplete — Phase 6.5 in `SKILL.md` exists specifically to close this.

## AI_TASK's own tool attachments live in `connections.tools[]`, not as separate `TOOL` nodes

Per `jelou workflow node-spec AI_TASK`: `configuration.connections.tools[]` is where an `AI_TASK` attaches callable tools for the model's own function-calling. Entries for **external marketplace tools** carry `toolId`, `toolkitId`, `version`, and — like a standalone `TOOL` node — point at a separate pullable backing workflow (Phase 6.5 applies the same way). Entries for **native platform tools** (`search_products`, `transfer_to_agent`, `send_interactive_message`, `get_current_date_time`, `get_weekday_of_a_date`, `send_call_to_action`, `crm_connect`, `memory_manager`, `flow_context`, `platform_reader`, `datum_records`) set `external: false` and have no backing workflow at all — they're platform code, out of scope, don't try to pull them.

**Consequence for counting "tools reviewed":** a project's true tool surface is standalone `TOOL` nodes *plus* every external entry in every `AI_TASK`'s `connections.tools[]`, deduplicated by `toolId`. Counting only standalone nodes undercounts — confirmed on Company E (reported 12, the project actually referenced 20+ once AI-agent-attached tools were counted).

## `contextBlocks.userInfo` structurally cannot inject the phone

An `AI_TASK` can enable `configuration.settings.contextBlocks.userInfo` (boolean `true`, or `{ enabled: true, fields?, title? }`) to auto-inject a "USER INFO" block into the model's context every turn. Per `jelou workflow node-spec AI_TASK`, the fields it can render are `id, names, referenceId, botId, roomId, socketId` — **`legalId` and `phone` are stripped as unsafe, unconditionally**, even if you list them in `fields`. There is no configuration that makes this block carry the phone.

This means: any `AI_TASK` with this block enabled is structurally exposed to `id` (a possibly-unreliable BSUID) with **no equivalent toggle to swap in `phone`** — unlike a plain `{{$user.id}}` mention in `instructions` text, this isn't a find-and-replace fix. The only way to give that model the resolved phone is to stop relying on this block for it and thread the value into `instructions` explicitly (e.g. `{{$user.phone}}` or the guard's memory variable, mentioned in the prompt text) — which is a prompt edit, so it's reported per Rule 7, never auto-rewritten.

## ID format for authored nodes/edges

Every node/edge id must be a 21-character string from `[A-Za-z0-9_-]` (or a `tmp_*` placeholder). Generate with any secure random generator over that alphabet — `jelou workflow validate` rejects anything else with `ids_invalid_node_id_format`/`ids_invalid_edge_id_format`.
