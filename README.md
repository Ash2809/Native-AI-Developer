# AI-Native Dev Pipeline

A production-ready, intent-centric software development pipeline powered by LangGraph and Google Gemini. Express requirements in natural language — the system synthesizes architecture, generates code, validates quality, and enforces compliance automatically, with humans governing every critical decision.

---

## What It Does

You describe what you want to build in plain English. The pipeline handles everything else:

1. Parses your intent into a structured manifest
2. Scans all libraries for IP and license risk
3. Maps applicable compliance frameworks (GDPR, OWASP, HIPAA, PCI-DSS)
4. Synthesizes a scored architecture with trade-off analysis
5. Generates production-grade Python code with compliance inline comments
6. Optimizes the code proactively (secrets, type hints, error handling)
7. Runs a security scan (SAST + OWASP checks)
8. Produces human-readable explainability docs for auditors
9. Scores code quality against acceptance criteria
10. Generates an immutable audit trail with cryptographic digest
11. Presents everything to a human for final sign-off

Three HITL (Human-in-the-Loop) gates pause the pipeline at critical decisions — no deployment happens without explicit human approval.

---

## Project Structure

```
pipeline/
├── main.py                  ← FastAPI entry point
├── .env                     ← API keys (create this)
├── requirements.txt
│
├── core/
│   ├── config.py            ← LLM setup, all constants and schemas
│   ├── state.py             ← DevState TypedDict, initial_state()
│   └── utils.py             ← extract_json, make_audit_entry, compute_hash
│
├── agents/
│   ├── intent.py            ← Parses NL input into structured manifest
│   ├── ip_guard.py          ← License and IP risk scanning
│   ├── compliance.py        ← Regulatory framework mapping
│   ├── architecture.py      ← Architecture synthesis and scoring
│   ├── codegen.py           ← Production code generation
│   ├── optimizer.py         ← Proactive code optimization
│   ├── security.py          ← SAST + OWASP security scanning
│   ├── explainability.py    ← Human-readable docs for auditors
│   ├── quality.py           ← Quality scoring against acceptance criteria
│   └── audit.py             ← Immutable audit trail + final sign-off note
│
├── graph/
│   ├── builder.py           ← LangGraph construction and compilation
│   └── routes.py            ← All conditional routing functions
│
└── api/
    ├── models.py            ← Pydantic request/response models
    ├── ws_manager.py        ← WebSocket connection manager
    ├── runner.py            ← Background pipeline execution
    └── routes.py            ← All FastAPI endpoints
```

---

## Prerequisites

- Python 3.11+
- Google Gemini API key
- pip

---

## Installation

```bash
# 1. Clone or copy the project
cd pipeline

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate      # Mac/Linux
venv\Scripts\activate         # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Create your .env file
echo "GEMINI_API_KEY=your_key_here" > .env
```

---

## Requirements

Create `requirements.txt` with:

```
fastapi
uvicorn
python-dotenv
langchain-google-genai
langgraph
pydantic
```

---

## Running the Server

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Stop the server: `Ctrl + C`

Force stop if needed:
```bash
pkill -f uvicorn
```

API docs available at: `http://localhost:8000/docs`

---

## API Reference

### Start a Pipeline

```http
POST /pipeline/start
Content-Type: application/json

{
  "raw_input": "Build a FastAPI web app with login and dashboard using PostgreSQL"
}
```

**Response:**
```json
{
  "thread_id": "abc-123",
  "status": "started",
  "ws_url": "/ws/pipeline/abc-123"
}
```

---

### Submit a HITL Decision

Called once per gate. The pipeline pauses at each gate and waits for this.

```http
POST /pipeline/{thread_id}/decide
Content-Type: application/json

{
  "choice": "A",
  "approver": "ash",
  "role": "Tech Lead",
  "feedback": null,
  "extra_notes": null,
  "justification": null,
  "risk_acknowledged": true
}
```

**Choice options:**
| Choice | Meaning |
|--------|---------|
| `A` | Approve — proceed to next stage |
| `R` | Reject — send back with feedback |
| `M` | Modify — approve with notes/changes |
| `H` | Hold — escalate, stop pipeline |

**Gates:**
| Gate | Pauses After | Reviews |
|------|-------------|---------|
| `hitl_gate_1` | IP scan | Intent manifest + IP clearance |
| `hitl_gate_2` | Architecture | Architecture + compliance gaps |
| `hitl_gate_3` | Audit | Security findings + quality score + full audit |

---

### Get Pipeline State

```http
GET /pipeline/{thread_id}/state
```

Returns current status, which gate it's paused at, audit log so far.

---

### Get Full Result

```http
GET /pipeline/{thread_id}/result
```

Returns everything once the pipeline is complete:
- `intent_manifest` — structured requirements
- `architecture` — selected pattern, layers, infrastructure
- `generated_code` — all files with code, rationale, compliance controls
- `security_report` — findings, OWASP refs, pass/fail
- `quality_report` — score, acceptance criteria check, recommendations
- `explainability_docs` — decision log, glossary, audit narrative
- `audit_log` — immutable timestamped chain
- `immutable_digest` — MD5 hashes of all pipeline outputs
- `sign_off_note` — paragraph written for regulators/auditors

---

### Cancel a Pipeline

```http
DELETE /pipeline/{thread_id}
```

---

### List All Pipelines

```http
GET /pipelines
```

---

## WebSocket Events

Connect to `ws://localhost:8000/ws/pipeline/{thread_id}` for real-time updates.

| Event type | When | Payload |
|------------|------|---------|
| `node_completed` | After each agent finishes | `node`, `summary`, `timestamp` |
| `hitl_pause` | Pipeline paused at a gate | `paused_at`, `gate_context`, `audit_log` |
| `pipeline_complete` | Pipeline finished | `final_status`, `sign_off_note`, `immutable_digest` |
| `error` | Something went wrong | `message` |
| `ping` | Keep-alive every 60s | — |

**JavaScript example:**
```javascript
const ws = new WebSocket(`ws://localhost:8000/ws/pipeline/${thread_id}`)

ws.onmessage = (e) => {
  const event = JSON.parse(e.data)

  if (event.type === 'node_completed') {
    console.log(`Completed: ${event.node}`, event.summary)
  }
  if (event.type === 'hitl_pause') {
    // show review UI with event.gate_context
    console.log(`Paused at: ${event.paused_at}`)
  }
  if (event.type === 'pipeline_complete') {
    console.log(`Done: ${event.final_status}`)
  }
}
```

---

## Full Frontend Flow

```javascript
// 1. Start pipeline
const { thread_id, ws_url } = await fetch('/pipeline/start', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ raw_input: 'Build a FastAPI app...' })
}).then(r => r.json())

// 2. Connect WebSocket for real-time events
const ws = new WebSocket(`ws://localhost:8000${ws_url}`)

// 3. When hitl_pause fires — show review UI and submit decision
await fetch(`/pipeline/${thread_id}/decide`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    choice: 'A',
    approver: 'ash',
    role: 'Tech Lead',
    risk_acknowledged: true
  })
})

// 4. Repeat for gate 2 and gate 3
// 5. When pipeline_complete fires — fetch full result
const result = await fetch(`/pipeline/${thread_id}/result`).then(r => r.json())
```

---

## HITL Decision Guide

### Gate 1 — Intent + IP Review
Review the parsed intent manifest and IP scan results.
- Use `R` with feedback if the tech stack is wrong (e.g. "use PostgreSQL not MongoDB")
- Use `M` with notes to add constraints without restarting
- Use `A` if everything looks correct

### Gate 2 — Architecture + Compliance Review
Review the architecture pattern, layers, and compliance gap resolution.
- Must type `ACKNOWLEDGE` to accept residual risks before approving
- Use `R` with feedback to change the architecture pattern or add layers
- Use `M` with feedback to inject architecture constraints (stored as `arch_feedback`)

### Gate 3 — Final Deploy Sign-off
Review security findings, quality score, and full audit trail.
- Approving with blockers requires a written justification
- Critical findings require typing `ACCEPT RISK` to formally acknowledge
- Use `H` to escalate to a senior reviewer without rejecting

---

## Compliance Coverage

The pipeline automatically detects and enforces:

| Framework | Triggered By |
|-----------|-------------|
| GDPR | user data, login, email, authentication |
| OWASP Top 10 | web, api, dashboard, jwt, password |
| HIPAA | health, medical, patient, diagnosis |
| PCI-DSS | payment, card, billing, stripe |

---

## Security Checks

The security agent runs both local regex scans and LLM-powered deep review:

| Rule | Severity |
|------|----------|
| Hardcoded secrets / API keys | Critical |
| SQL injection patterns | Critical |
| Debug mode enabled | High |
| HTTP instead of HTTPS | High |
| Unlicensed imports | High |
| Bare except clauses | Medium |

The pipeline retries codegen up to 3 times if security fails, then forces forward with findings documented in the audit report.

---

## Audit Trail

Every pipeline run produces:
- A timestamped audit log entry for every agent and HITL decision
- Named approvers with roles at each gate
- An immutable MD5 digest of the intent, architecture, code, and audit chain
- A single-paragraph sign-off note written for regulators and external auditors

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GEMINI_API_KEY` | Yes | Google Gemini API key |

---

## Troubleshooting

**Server won't start:**
```bash
# check port is free
lsof -i :8000
# kill if occupied
kill -9 <PID>
```

**JSON parse errors from LLM:**
These are handled gracefully — the optimizer skips on parse failure and the pipeline continues. If they're frequent, the model may be hitting context limits on large codebases.

**Pipeline stuck after gate decision:**
Check that the WebSocket is connected before calling `/decide`. The resume event fires immediately on decision — if the WS disconnects first, reconnect and poll `/pipeline/{id}/state` to check progress.

**`pipeline_stopped` not working:**
Ensure `DevState` includes `pipeline_stopped: bool` and `initial_state()` sets it to `False`. All agents check `should_stop(state)` at the top.
