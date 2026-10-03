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
| **Requests** (home) | Live stream of prompts, answers and tool calls with status: Allowed, Redacted, Blocked, Throttled, Monitored. Summary row, search, filters, "Possible false positives" quick filter. |
| **Request detail** | Plain-language reason, session context (previous messages and tool calls), what was sent vs what the model received, full inspection chain with latency and AI confidence. Actions: mark as false positive, add exception, replay against current or draft policy, open node in builder. |
| **Policy builder** | One company-wide policy as a top-to-bottom pipeline of nodes (rules, AI checks, access, signature feed, budgets). Node library, drag to reorder, per-node settings, agent-specific thresholds, exceptions, validation. Side-by-side YAML editor with two-way sync. External edits to `policy.yaml` are flagged on the changed nodes; conflicts with unsaved edits are resolved in a dialog. |
| **Policy history** | Every version with author, time and source (console, policy.yaml, rollback). Pipeline diff and YAML diff. Rollback creates a new version, history is never deleted. |

## Demo script (2 minutes)

1. **Requests** → turn on *Possible false positives* → open the *red-team training* request (blocked at confidence 0.81, threshold 0.80).
2. **Add exception** → *Save and apply* → Replay shows *Blocked → Allowed*.
3. **Policy history** → the new version with author and diff.
4. **Policy builder** → *Simulate an edit to policy.yaml* → banner and badges on changed nodes. With unsaved edits, the conflict dialog appears.

## Integration points for the backend

The prototype keeps all state in memory (`S` in the script). To connect a real backend, replace:

1. **Request stream**: the `setInterval` mock in "live stream" → SSE/WebSocket from the control layer, plus `GET /requests` for the initial list.
2. **Policy**: `S.live` → `GET /policy`; `commit()` → `PUT /policy` (returns the new version).
3. **History**: `S.history` → `GET /policy/versions`, `POST /policy/rollback/{version}`.
4. **Replay**: `evaluate()` → `POST /replay` with request id and optional draft policy.
5. **External file changes**: `simulateFileEdit()` → a `policy_changed` event from the backend file watcher.

Each request event needs: id, timestamp, user, agent, direction, target, text, decision, triggering node, policy version, and the check chain (`node id, kind, result, latency_ms, confidence, threshold`).

## Tech

Plain HTML, CSS and JavaScript in one file. [js-yaml](https://github.com/nodeca/js-yaml) for YAML parsing. Dark theme by default, light theme optional.
