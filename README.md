# Interruptible AI Voice Browser Agent — Full Integration

This is the complete Lead Integration Engineer deliverable: React frontend,
FastAPI backend, LLM orchestration, central task versioning, and the
WebSocket wiring connecting Person 1's voice module and Person 2's browser
module. Neither of those modules was rewritten — see `backend/voice/` and
`backend/browser/`, both used strictly through their existing interfaces.

```
backend/
  main.py              FastAPI app: WebSocket /ws + REST endpoints
  task_manager.py       Central task versioning (THE most important file)
  agent.py              LLM orchestration -> constrained structured intent
  events.py              WebSocket event protocol builders
  models.py              TaskState enum, Task/ParsedIntent/Constraints
  test_integration.py    Covers the spec's 6-test checklist
  requirements.txt
  .env.example
  voice/                 Person 1's module, unmodified
  browser/               Person 2's module, unmodified

frontend/
  src/
    App.jsx              Wires WebSocket events to UI state
    main.jsx
    api.js                REST client
    websocket.js           WebSocket client
    components/
      VoiceButton.jsx
      StatusIndicator.jsx
      Transcript.jsx
      Results.jsx
      TaskTimeline.jsx
  index.html, package.json, vite.config.js, tailwind.config.js, postcss.config.js
```

---

## 1. Setup commands

### Backend
```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
playwright install chromium
cp .env.example .env
```

### Frontend
```bash
cd frontend
npm install
```

---

## 2. Environment variables

All in `backend/.env` (copy from `.env.example`). Every one of these has a
**free, zero-setup default** — nothing here requires an API key to run the
full interrupt/versioning demo:

| Variable | Default | Notes |
|---|---|---|
| `STT_ENGINE` | `mock` | `whisper_local` (free, local) or `gemini_api` for real transcription |
| `RIME_API_KEY` | placeholder | leave as-is for mock audio; set a real key from app.rime.ai/tokens for real speech |
| `LLM_ENGINE` | `mock` | zero-cost rule-based intent extraction; set to `gemini` for a real LLM call |
| `LLM_MODEL` | `gemini-2.5-flash` | only used when `LLM_ENGINE=gemini` |
| `HEADLESS` | `false` | keep `false` so the browser is visible during the demo |
| `HOST` / `PORT` | `127.0.0.1` / `8000` | match whatever actually binds on your machine |

Frontend: optionally set `VITE_API_URL` / `VITE_WS_URL` in `frontend/.env` if the backend isn't on `localhost:8000`.

---

## 3. Running frontend and backend simultaneously

**Terminal 1 — backend:**
```bash
cd backend
venv\Scripts\activate   # or: source venv/bin/activate
python main.py
```
Look for `Application startup complete.`

**Terminal 2 — frontend:**
```bash
cd frontend
npm run dev
```
Open the printed URL (usually `http://localhost:5173`).

**Terminal 3 (optional) — run the test checklist:**
```bash
cd backend
python test_integration.py
```

---

## 4. Architecture (matches the spec exactly)

```
React  →  FastAPI (/api/voice-command or /ws)  →  TaskManager
                                                       ↓
                                              agent.py (LLM/mock)
                                                       ↓
                                          browser.search_products()  (Person 2)
                                                       ↓
                                              rime_service.speak()   (Person 1)
                                                       ↓
                                                    React (via WS)
```

**Interruption path:**
```
Mic → STT (Person 1) → FastAPI → agent.parse_utterance() (detect new instruction)
   → TaskManager invalidates current_task_id → voice_manager.interrupt_current_task()
   → browser.cancel_task(old_id) → new task_id created
   → browser.search_products(new_query, new_id) → old results ignored (is_current() check)
   → new results returned → rime_service.speak(new result) → React updates via WS
```

---

## 5. Task versioning — the core logic

`task_manager.py`'s `TaskManager` holds exactly one `current_task_id`. Every
async step (`search_products`, speaking) checks `self.is_current(task_id)`
immediately before storing or broadcasting anything. If the check fails,
the result is silently discarded — this is what guarantees a stale task's
late-arriving result can never reach the UI, no matter how the timing lands.

```python
if not self.is_current(new_task_id):
    return None  # discarded
```

---

## 6. API contracts

### `POST /api/voice-command`
Multipart form, field `file` = recorded audio blob.
```json
{ "transcript": "black running shoes under 2000", "result": { "...": "browser result dict, or null if discarded" } }
```

### `POST /api/text-command`
```json
{ "text": "black running shoes under 2000" }
```
Same response shape as above. Bypasses audio entirely — useful for demos where the mic might misbehave on stage.

### `POST /api/reset`
Clears all task state. Response: `{"status": "reset"}`

### `GET /api/state`
```json
{ "current_task_id": "task_xxxx", "task": { "...": "Task model, or null" }, "voice_state": "IDLE" }
```

---

## 7. WebSocket event specification (`ws://localhost:8000/ws`)

Client → server:
```json
{"action": "utterance", "text": "black running shoes under 2000"}
{"action": "reset"}
{"action": "start_listening"}
```

Server → client (broadcast to all connected clients):
```json
{"type": "transcript.updated", "task_id": "task_001", "text": "..."}
{"type": "task.started", "task_id": "task_002", "query": "...", "constraints": {"max_price": 1500, "size": "9", "color": "black"}}
{"type": "task.interrupted", "old_task_id": "task_001", "new_task_id": "task_002"}
{"type": "state.changed", "task_id": "task_002", "state": "BROWSING"}
{"type": "browser.started", "task_id": "task_002"}
{"type": "browser.result", "task_id": "task_002", "results": [{"name": "...", "price": "...", "url": "...", "image": "..."}]}
{"type": "speech.started", "task_id": "task_002", "text": "..."}
{"type": "task.completed", "task_id": "task_002"}
{"type": "task.error", "task_id": "task_002", "error": "..."}
```

`TaskState` values used in `state.changed`: `IDLE, LISTENING, THINKING, BROWSING, SPEAKING, INTERRUPTED, COMPLETED, ERROR`.

---

## 8. Integration instructions for Person 1 (voice)

Nothing to change. `task_manager.py` calls exactly the interface you exposed:
```python
from voice import voice_manager, rime_service
rime_service.speak(text, task_id)                                  # -> bytes | None
voice_manager.interrupt_current_task(new_task_id=..., reason=...)  # stops stale speech
```
If you rename or change the signature of either, update the two call sites in `task_manager.py` (`_interrupt` and `handle_utterance`) — nowhere else references your module.

## 9. Integration instructions for Person 2 (browser)

Nothing to change. `task_manager.py` calls exactly your interface:
```python
from browser import search_products, create_task, cancel_task
create_task(task_id, query)
result = await search_products(query, task_id)   # returns your standard result dict
cancel_task(old_task_id)
```
`agent.build_search_query()` is the only place that turns the LLM's structured `{query, constraints}` back into the plain-text string your `search_products()` expects — if you ever change what your function accepts, that's the one function to update.

---

## 10. Test checklist (see `test_integration.py` for automated versions of all six)

| # | Test | Expected |
|---|---|---|
| 1 | Normal voice search | Completes, `task.started` → `task.completed` events fire |
| 2 | Interrupt while AI is speaking | Old task discarded (`None` returned), new task delivers |
| 3 | Interrupt while browser is searching | Same guarantee, mid-browse timing |
| 4 | Old task result arrives after new task started | `is_current()` rejects it — never surfaces |
| 5 | New task completes normally (no interrupt) | No spurious `task.interrupted` event |
| 6 | API/search failure | Returns clean `error`/`completed_empty` dict, never raises |

Run all six automatically:
```bash
cd backend
python test_integration.py
```

---

## What was deliberately not built (per the spec)

Login, checkout, payment, multiple concurrent browser agents, persistent
memory, a database, authentication, a mobile app, or support for more than
one browsing workflow. This is a one-day hackathon MVP — the versioning
logic in `task_manager.py` is the part that matters, and it has no
shortcuts taken in it.
