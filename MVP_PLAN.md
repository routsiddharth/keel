# Keel MVP — Build Plan

**Goal:** the demo in BRIEF.md's "next week's target", and nothing else. One closed job goes through the block view, Keel's policy checks, and the audit log, and comes out as a verified equipment record in a mock field service system.

> **MVP boundary:** this is a controlled prototype for a recorded demo and fixture-based evaluation, not a production deployment. It is intentionally not crash-safe, adversarially isolated, authenticated, multi-user, concurrency-safe, or ready for customer data. Build only what is specified below.

**Done when:** we have a screen recording where:
1. A job closes in the mock FSM.
2. The agent picks it up, reads the notes and the nameplate photo, and extracts make, model, serial, and install date.
3. Confident fields get written. Uncertain fields go to an approval queue, and a human approves or corrects them.
4. The next maintenance date gets set.
5. From that run's log page, we click **Test blocked actions**. Keel attempts one invoice read and one customer-message send through the same gateway used by blocks; both are denied, appear in the timeline, and never reach the mock FSM.
6. The audit log shows every step, and the block view reads as plain English.

We also need an eval script that scores 15–20 fixture jobs against the BRIEF's "working" criteria. The recording is the demo. The eval is how we know the demo isn't a fluke.

---

## 1. Scope

### In
| Piece | MVP form |
|---|---|
| Field service system | A mock "FSM" (fake ServiceTitan) running as its own service, with its own API and credentials |
| Keel OS | A gateway that holds the FSM credentials, checks each action against a policy (allow / deny / require approval), and writes an audit log |
| Blocks | 5 hand-built blocks, enough for the wedge task |
| Agent | One agent, hand-authored as a JSON spec (we operate it ourselves, per BRIEF rollout) |
| Builder | **Read-only** block view of that spec: the palette, filtered by permissions |
| Approvals | A queue page where the office manager approves, edits, or rejects uncertain fields |
| Audit | One append-only event stream and a log page showing block execution, model completion, approvals, and every gateway decision/result |
| Eval | A script plus fixtures with ground truth |

### Out (deliberately)
- **Plain-English → agent generation.** We write the spec by hand. The BRIEF says we operate it ourselves first, and the thing to test is whether the owner can *read* an agent, which a hand-written spec tests just as well.
- **Drag-and-drop editing.** View only.
- **Real ServiceTitan/Jobber/Housecall Pro.** The connector sits behind the gateway, so we can swap it in later without touching the blocks.
- **Auth, multi-user, multi-tenant.** One hardcoded owner ("Mike, owner") and one office manager. `on_behalf_of` is still passed and checked so the model is right, but there's no login.
- **Self-host packaging, update delivery, pricing.**
- **Retries, scheduling, notifications.** Failure means stop, log it, and show it.
- **General LLM agent loop.** See §3. The model never picks tools.

---

## 2. Open questions, answered *for the MVP only*

These are defaults so we can build. They aren't final decisions (README "Open" list).

| Question | MVP answer |
|---|---|
| Which systems first? | One mock FSM, shaped like ServiceTitan's job, customer, and equipment objects |
| Where does the model show up? | Only inside **judgment blocks**, which look different in the view (a distinct color and a "Keel decides" label). Deterministic blocks look like plain actions. We want to learn whether the owner notices the difference and cares. |
| How do agents start? | Trigger block "When a job is closed", which polls the mock FSM every 10s. There's also a "Run on this job" button for the demo. |
| Fails halfway? | Stop the run, mark it `failed`, and log the error. Writes that already happened stay, and they're in the log. No retry. |
| Try before live? | `dry_run` flag: the gateway runs policy checks and logs everything but doesn't execute writes. It's cheap to add and it's the probable long-term answer, so we build it now. |
| Audit trail for? | Debugging, plus owner verification. Not compliance. |

---

## 3. Architecture

```
            ┌──────────── Keel (one Node process) ─────────────┐
 Mock FSM   │                                                  │
 (separate  │  Runner ──► Block impls ──► Gateway ──► Connector│──► Mock FSM API
  process,  │    │            │             │  policy check    │    (holds the only
  own API   │    │            │             │  audit append    │     credentials)
  key)      │    │            └► Claude (judgment blocks only) │
            │    ▼                          ▼                  │
            │  SQLite: runs, approvals, audit_log, agents      │
            │  Web UI: block view · approvals · audit log      │
            └──────────────────────────────────────────────────┘
```

**Core rule:** block implementations never hold credentials and never call the FSM directly. Every external effect goes through `gateway.call(ctx, action, args)`. That call is the permission boundary, and it's where the audit log is written. If it's worth demoing, it's worth enforcing structurally, not by convention.

**The agent is a fixed pipeline, not an LLM loop.** The runner executes blocks in order. The model is called inside the "Extract equipment details" block as a pure function: notes and photo in, structured fields plus confidence out. It gets no tools, so it can't take an action that isn't a block. This is README's "can't, not shouldn't" in its most literal form.

### Stack (pick and don't revisit)
- **TypeScript / Node 22**, one repo, one `package.json`
- **Hono** for both HTTP servers (Keel, and the mock FSM on a separate port)
- **SQLite** via `better-sqlite3`, one file per service
- **Server-rendered HTML + htmx** for the UI. No React build step.
- **Claude API** (vision + structured JSON output) for extraction
- **Vitest** for tests and the eval runner

---

## 4. Data model

### Mock FSM (`fsm.db`)
- `customers(id, name, address, phone)`
- `jobs(id, customer_id, status, closed_at, tech_name, closeout_notes, nameplate_photo_path)`
- `equipment(id, customer_id, job_id UNIQUE, type, make, model, serial, install_date, next_maintenance_due, updated_at)`
- `invoices(id, job_id, amount)`: exists **so the agent can be denied access to it**
- `messages(id, customer_id, body)`: same reason

Mock FSM API (bearer token; Keel's connector is the only holder):
- `GET /jobs?status=closed&since=`
- `GET /jobs/:id`
- `GET /jobs/:id/photo`
- `GET /customers/:id`
- `PUT /customers/:customerId/equipment/by-job/:jobId`: partial upsert for the single equipment record associated with the job
- `GET/PUT /invoices/:id`, `POST /customers/:id/messages`: the forbidden ones
- `POST /jobs/:id/close`: the demo control that simulates a tech closing a job

### Keel (`keel.db`)
- `agents(id, name, spec_json, granted_actions_json, created_by)`
- `users(id, name, role, allowed_actions_json)`: 2 seeded rows
- `runs(id, agent_id, job_id, on_behalf_of, status[running|waiting_approval|succeeded|failed], dry_run, started_at, ended_at)`
- `approvals(id, run_id, field, proposed_value, evidence, reason, status[pending|approved|edited|rejected], final_value, decided_by, decided_at)`
- `audit_log(id, ts, run_id, agent_id, on_behalf_of, block_id, event_type, operation_id, action, args_json, decision, result_summary, error)`, **append-only** (no update/delete code path; add a SQLite trigger that raises on UPDATE/DELETE)

### Audit event contract

The audit table is one event stream for both the readable run timeline and the gateway proof. Fields that do not apply to an event are nullable. `operation_id` is a UUID created once per `gateway.call`; it correlates its decision and result rows.

The MVP writes these events:

| `event_type` | Written by | Required contents |
|---|---|---|
| `block_started` | runner, before each block | `block_id` |
| `block_completed` | runner, after each block | `block_id`, short `result_summary` |
| `block_failed` | runner, when a block throws | `block_id`, `error` |
| `model_completed` | extraction block, after Claude returns | `block_id`, extracted values, confidence and source in `result_summary`; do not copy the image into the log |
| `approval_requested` | runner, once per uncertain field | `block_id`, field, proposed value and reason in `args_json` |
| `approval_decided` | approval route, once per decision | `block_id`, field, outcome, final value and `decided_by` in `result_summary` |
| `gateway_decision` | gateway, before any connector call | `operation_id`, `action`, redacted `args_json`, `decision` |
| `gateway_result` | gateway, after the decision is handled | same `operation_id` and `action`, plus result or error |

Every started block gets exactly one terminal `block_completed` or `block_failed` event. Every `gateway.call` gets exactly one `gateway_decision` and one `gateway_result`. For a denial, the result says `not executed: denied`; for dry-run, it says `not executed: dry run`. Only an allowed, non-dry-run decision invokes the connector.

The run page groups the two gateway rows by `operation_id` into one timeline item, so the owner sees a single action with its decision and result rather than implementation-level noise.

---

## 5. Policy

Actions are named strings, e.g. `fsm.jobs.read`, `fsm.equipment.write`, `fsm.invoices.read`, `fsm.messages.send`.

Effective permission for a call = **agent grant ∩ user permission**, then a per-action rule:

```yaml
# policy.yaml
rules:
  fsm.jobs.read:        allow
  fsm.customers.read:   allow
  fsm.equipment.write:  allow          # per-field approval handled by runner (see §6)
  fsm.invoices.*:       deny
  fsm.messages.*:       deny
default: deny
```

Gateway algorithm:
1. Action not in `agent.granted_actions`, or not in the user's `allowed_actions` → **deny**
2. Look up the rule, falling back to `default` → allow / deny / require_approval
3. Generate an `operation_id` and append `gateway_decision` before executing
4. If allowed and not `dry_run`, execute via the connector
5. Append `gateway_result` with the same `operation_id`; never update the decision row

The palette shown in the block view = blocks whose `requires` actions are all permitted for the viewing user. That's BRIEF's "palette only shows actions the owner's permissions allow".

### Demo-only policy test

The normal five-block pipeline has no reason to touch invoices or messages, so the demo needs an explicit way to prove the gateway denies them:

- `POST /runs/:id/demo-policy-check` is a hardcoded, demo-only route. It is not a sixth agent block and does not appear in the palette.
- The route loads the selected run and constructs the same `ctx` used by that run (`run_id`, `agent_id`, `on_behalf_of`, `dry_run: false`). It does not accept an action or identity from the browser.
- It calls the gateway twice with `block_id: "demo.policy_check"`: `fsm.invoices.read` for the run's job and `fsm.messages.send` for the job's customer.
- Neither action is in the agent grant, so both calls must return a structured denied result, write correlated `gateway_decision` and `gateway_result` events, and make zero connector requests.
- The route returns an htmx fragment that refreshes the timeline. The UI labels these rows **Demo policy test (not part of the agent)** so the recording does not imply the agent unexpectedly attempted unrelated work.

---

## 6. The blocks

Each block is `{ id, label (plain English), kind: trigger|action|judgment, requires: [actions], run(ctx, input) → output }`.

| # | Block (as the owner reads it) | Kind | Requires |
|---|---|---|---|
| 1 | **When a job is closed** | trigger | `fsm.jobs.read` |
| 2 | **Read the job's close-out notes and nameplate photo** | action | `fsm.jobs.read` |
| 3 | **Figure out the equipment's make, model, serial number, and install date** | judgment | none (model call only) |
| 4 | **Save the equipment details to the customer's record** (asks the office manager about anything Keel isn't sure of) | action | `fsm.equipment.write` |
| 5 | **Set the next maintenance date to [12] months after this job** | action | `fsm.equipment.write` |

The agent spec is just:
```json
{ "name": "Equipment record from closed jobs",
  "blocks": [
    {"block": "trigger.job_closed"},
    {"block": "fsm.read_job"},
    {"block": "judge.extract_equipment"},
    {"block": "fsm.save_equipment", "params": {"ask_if_unsure": true}},
    {"block": "fsm.set_next_maintenance", "params": {"months": 12}}
  ] }
```

### Block 3: extraction and "unsure" (the one to get right)
The Claude call gets the close-out notes and the nameplate image and must return, per field:
`{ value | null, confidence: high|low, source: photo|notes|both, evidence: "short quote or what's visible" }`

A field counts as **uncertain**, and goes to approval rather than being written, if *any* of these hold:
- the model returned `low` or `null`
- photo and notes both mention it and **disagree**
- deterministic validation fails: install date doesn't parse or is in the future, serial is <5 chars or has characters that aren't alphanumeric/dash, make isn't in a known-manufacturer list (~30 HVAC brands)

Install date is often missing from the plate. The MVP does **not** decode dates from serial numbers (manufacturer-specific; later). Missing means uncertain means ask.

### Block 4: save with approvals
- Write the confident fields immediately through the gateway.
- For each uncertain field, create an `approvals` row and set the run to `waiting_approval`.
- When all approvals for the run are decided, the runner resumes: it writes the approved or edited values (again through the gateway, logged with `decided_by`) and continues to block 5.
- Rejected fields stay empty in the FSM. That's the correct outcome, not a failure.

### Exact equipment behavior for the MVP

The wedge assumes exactly one HVAC equipment unit per job. Every fixture must contain at least one usable identifying value after approvals; multi-unit jobs and completely unidentified equipment are out of scope.

- Equipment identity is `job_id`. The mock FSM enforces `UNIQUE(equipment.job_id)` and generates a stable `equipment.id` on the first upsert.
- `PUT /customers/:customerId/equipment/by-job/:jobId` first verifies that the job belongs to the customer. A mismatch returns `409` and the block fails.
- The request body is a partial patch containing only `make`, `model`, `serial`, `install_date`, `next_maintenance_due`, and the fixed MVP value `type: "hvac"`. Omitted fields retain their current values; rejected fields are omitted and remain `NULL`.
- The first Block 4 call upserts only confident, non-null fields. After approval, a second call patches only approved or edited values. Re-running either patch produces the same record rather than another equipment row.
- Block 5 reads `jobs.closed_at`, takes its UTC calendar date, adds 12 calendar months, and writes that `YYYY-MM-DD` as `next_maintenance_due`. Preserve the month and day when valid; clamp leap-day `2028-02-29` to `2029-02-28`.
- Block 5 updates the same `job_id`-keyed equipment row. Its gateway result summary includes the equipment ID and computed due date so the log and eval can assert them.

---

## 7. UI (3 pages, server-rendered, deliberately plain)

1. **Agent** (`/agents/:id`): the block view. Vertical stack of cards, one per block, each with its plain-English label and a small "can touch: Jobs (read), Equipment (write)" line. Judgment blocks are visibly different. A sidebar shows the palette (blocks this user is allowed to use) and, greyed out, "Keel cannot: see invoices · message customers". Buttons: "Run on job…" and a dry-run toggle.
2. **Approvals** (`/approvals`): one card per pending field. It shows the nameplate photo, the notes with the evidence highlighted, the proposed value, and why Keel is unsure. Actions: Approve / Edit & approve / Leave blank.
3. **Log** (`/runs/:id` and `/log`): a timeline per run built from the audit event contract above. Block events, model completion, approval activity, and grouped gateway operations appear in order. Each gateway item shows time, block label, action, decision badge, and result; denied items are red. `/runs/:id` also has a **Test blocked actions** button that posts to `/runs/:id/demo-policy-check` and refreshes the timeline.

Plus a tiny **mock FSM admin** page (`:4001/`) listing jobs with "Close job" buttons and the equipment table, so the recording can show the record changing in "their" system.

---

## 8. Fixtures and eval

**Fixtures** (`fixtures/jobs/*.json` + photos): 15–20 jobs, each with notes, a nameplate photo, and `expected` values (or `expected: "uncertain"` for fields a human should be asked about).

Mix:
- ~8 clean: clear plate, notes agree
- ~3 blurry or partly occluded plate
- ~3 where notes contradict the plate (tech typo in serial)
- ~3 with no install date anywhere
- ~2 notes-only (no photo)

Photos: real nameplate photos from the design-partner HVAC owner if we can get them this week. Otherwise public images of nameplates. Notes: we write them, in a tech's voice (terse, abbreviations, typos).

**Eval script** (`npm run eval`): resets both DBs, seeds fixtures, runs the agent on every job with an auto-approver that answers from ground truth, then scores each run against the BRIEF's five "working" criteria:

| Criterion | Check |
|---|---|
| Fields match ground truth | FSM equipment row == `expected` for each field |
| Correct customer and job | equipment row's `customer_id` / `job_id` match the fixture |
| No out-of-policy action | zero `allow` decisions on actions outside the grant (plus the red-team test below) |
| Uncertain → human, never guessed | every field marked `uncertain` in the fixture produced an approval row. **Any wrong value written without approval is a hard fail.** |
| Every call logged | every block has one start and terminal event; every gateway operation has one decision and result; allowed non-dry-run operations equal instrumented connector requests; denied operations make no connector request |

Output: a per-job pass/fail table and a headline "N/20 verified". That's the metric's pre-launch stand-in.

**Red-team test** (Vitest): call the same demo policy-check service used by `POST /runs/:id/demo-policy-check`. Assert the invoice read and message send are denied, each has correlated decision/result events, and neither reaches the mock FSM (the mock FSM counts requests). The HTTP route gets a thin integration test proving the button invokes that service for the selected run.

The single most important number is **wrong values written without approval = 0**. Fewer correct auto-fills is acceptable. A confident wrong serial number in the system of record is the failure that kills trust.

---

## 9. Build order

Two people, split along the gateway boundary. **A** owns the OS side, **B** owns the agent and UI side. Each step ends with something runnable.

| Day | A: OS / infra | B: Agent / UI |
|---|---|---|
| 1 | Repo scaffold, mock FSM service + DB + seed + API + admin page | Collect/produce fixtures (photos + notes + ground truth). Start today, it's the long pole. |
| 2 | Gateway: policy.yaml, grant ∩ user check, append-only audit events, connector, dry-run, and demo policy-check service/route. Red-team test passing. | Extraction block alone: Claude call + validation + uncertainty rules, run as a script over fixtures. Iterate on the prompt here. |
| 3 | Runner: block registry, sequential execution, run states, pause/resume on approvals, trigger polling | Blocks 2, 4, 5 on top of the gateway |
| 4 | Eval script + scoring table | UI: block view, approvals page, log page |
| 5 | Fix whatever the eval exposes. Tune uncertainty thresholds until "wrong without approval" = 0. | Polish the block copy: read it to someone non-technical and see if they can say what it does. Record the demo. |

**Checkpoint at end of day 2:** the extraction script's accuracy on fixtures. If clean-plate accuracy is under ~90%, spend day 3 on the prompt and image handling (crop/rotate, higher resolution) before building more around it. Everything downstream is only as good as this block.

---

## 10. Repo layout

```
keel/
  src/
    fsm/            # mock field service system (server, db, seed, admin page)
    os/             # gateway.ts, policy.ts, audit.ts, connector.ts
    blocks/         # one file per block + registry.ts
    runner/         # runner.ts, trigger poller
    ui/             # hono routes + htmx templates
    llm/            # claude client, extraction prompt, schema
  policy.yaml
  agents/equipment-record.json
  fixtures/jobs/    # *.json + photos/
  eval/run-eval.ts
  test/             # gateway, red-team, runner, validation
```

---

## 11. Risks for this week

- **No real data.** This is the stated blocker. Fallback: public nameplate images + our own notes. The demo still works; the eval just proves less. Keep asking the HVAC owner for 10 photos from past jobs, which is a far smaller ask than API access.
- **Extraction accuracy on bad photos.** Mitigated by design: bad photos should produce approvals, not guesses. Measure "wrong without approval", not raw accuracy.
- **Scope creep toward the Builder.** Editing, NL generation, and drag-and-drop are all out. The Builder test this week is only: *can someone read the block view and say what the agent will do?* Try it on one non-technical person and write down what they say.

## 12. After the MVP (not now, just so it's written down)
Real connector for whichever FSM the design partner uses → hand-edit blocks in the UI → plain-English generation → owner reads/approves agents unaided → self-host packaging.
