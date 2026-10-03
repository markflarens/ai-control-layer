# AI Control Layer: Control Panel (frontend)

HackYeah 2026, "AI Control Layer" challenge.

Admin console for an AI control layer that sits between company AI agents and AI models (LLMs, MCP tools, APIs). It lets a security admin watch every request, investigate why a request was blocked, and edit the central policy as a visual pipeline that stays in sync with `policy.yaml`.

This repository currently contains the **frontend prototype**. The backend (proxy, guardrails, policy engine) is built separately. All data in the prototype is mocked.

## Run it

No build step. Open `frontend/index.html` in a browser, or serve the folder:

```bash
npx serve frontend
# or
python3 -m http.server -d frontend 8080
```

Requires internet access for Google Fonts and js-yaml (loaded from cdnjs).

## Screens

| Screen | What it does |
|---|---|
| **Requests** (home) | Live stream of prompts, model answers and service calls with status: Allowed, Redacted, Blocked, Throttled, Monitored. Search, filters, "Possible false positives". |
| **Request detail** | Plain-language reason, session context, what was sent vs received, full inspection chain (matched rule, AI label, latency). Actions: mark as false positive, add exception, create or open the access rule, save as test case, replay. |
| **Services** | Catalog of internal services agents can call. Add a service by importing its schema (actions and typed parameters), verify it against the running service, see which rules cover each action. |
| **Policy builder: Checks** | Global pipeline: deterministic rules (secrets, personal data, custom patterns), access (allowed models, denied paths), service access stage, signature feed, budgets, AI classification. Add checks with the + buttons anywhere in the pipeline. AI checks answer with one label; each label has its own reaction, plus reactions for unknown labels and timeouts. |
| **Policy builder: Service rules** | Who → service → action → conditions on parameters → Allow or Block. The most specific rule wins; on a tie, Block wins. Calls without a rule use the default. |
| **Policy builder: Test cases** | Requests with an expected result, run against the live version and the draft without calling real services. |
| **Check impact** | Before applying, every test case runs on the live and the new version. New mismatches block applying until the admin confirms. |
| **History** | Every version with author, time and source (console, policy.yaml, rollback). Change list and YAML diff. Rollback goes through the same impact check. |
| **Training data** | Results of AI checks kept as examples (input after redaction, model label, human review, corrected label, model and instruction version, latency, cache). Review and export as JSONL. |

The YAML editor in the builder works in both directions. External edits to `policy.yaml` are flagged on the changed checks and rules; conflicts with unsaved edits are resolved in a dialog.

## Demo script (3 minutes)

1. **Policy builder → Service rules** → rule `support-bot → documents.read` → change the action to *Any action* → *Check impact and apply*. The impact check shows that *Support bot cannot delete documents* is now allowed, so applying is blocked. Go back and fix the rule.
2. **Requests** → *Possible false positives* → the *red-team training* request blocked by the prompt injection classifier → *Add exception* (also saved as a test case) → the impact check shows 2 fixed tests → apply → Replay shows *Blocked → Allowed*.
3. **Services** → *Add service* → import the sample schema → verify → add a rule for the new service.
4. **Checks** → Prompt injection classifier → *Simulate a model failure: unknown label* → **Test cases** shows that requests fail closed.
5. **Training data** → confirm or correct a label → *Export reviewed examples*.
6. **Policy builder** → *Simulate an edit to policy.yaml* → banner and badges on the changed items. **History** → YAML diff and rollback.

## Integration points for the backend

The prototype keeps all state in memory (`S` in the script). To connect a real backend, replace:

1. **Request stream**: the `setInterval` mock in "live stream" → SSE/WebSocket from the control layer, plus `GET /requests` for the initial list.
2. **Policy**: `S.live` → `GET /policy`; `commit()` → `PUT /policy` (returns the new version).
3. **History**: `S.history` → `GET /policy/versions`, `POST /policy/rollback/{version}`.
6. **Services**: `S.services` → `GET /services`, `POST /services` (schema import), `POST /services/{id}/verify`.
7. **Test cases**: `S.tests` and `runTest()` → `GET/PUT /tests`, `POST /tests/run` with a draft policy.
8. **Training data**: `S.training` → `GET /training-examples`, `PATCH /training-examples/{id}` (review), `GET /training-examples/export`.
4. **Replay**: `evaluate()` → `POST /replay` with request id and optional draft policy.
5. **External file changes**: `simulateFileEdit()` → a `policy_changed` event from the backend file watcher.

Each request event needs: id, timestamp, user, agent, direction, target, text, decision, triggering node, policy version, and the check chain (`node id, kind, result, latency_ms, confidence, threshold`).

## Tech

Plain HTML, CSS and JavaScript in one file. [js-yaml](https://github.com/nodeca/js-yaml) for YAML parsing. Dark theme by default, light theme optional.
