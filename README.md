# Interruptible AI Voice Browser Agent

An end-to-end voice-controlled browser agent capable of browsing live e-commerce websites and synthesizing responses, featuring **zero-latency task interruption and central task versioning**.

When a user speaks while the agent is browsing or speaking, the agent immediately invalidates the stale task, cancels active browser actions, halts speech synthesis, and pivots to the new intent without audio collisions or out-of-order results.

---

## 🔗 Live Demo & Links

| | Link |
|---|---|
| 🌐 **Live Frontend** | https://interruptible-ai-voice-browser-agen.vercel.app/ |
| ⚙️ **Live Backend API** | https://interruptible-ai-voice-browser-agent.onrender.com |
| 🎥 **Demo Video (4–5 min)** | [_Coming soon — add your recorded demo link here_](#) |
| 📦 **Source Code** | (https://github.com/iamshrutijain11/Interruptible-AI-Voice-Browser-Agent) |

---


## 1. Setup Instructions

### Prerequisites
- Python 3.10+
- Node.js 18+ and npm
- Chromium (installed via Playwright)

### Backend Setup
```bash
cd backend
python -m venv venv

# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
playwright install chromium
cp .env.example .env
```

### Frontend Setup
```bash
cd frontend
npm install
```

---

## 2. Running Frontend and Backend Simultaneously

### Terminal 1 — Backend (FastAPI + WebSocket Server)
```bash
cd backend
# Activate venv first
venv\Scripts\activate      # Windows: venv\Scripts\activate | Unix: source venv/bin/activate
python main.py
```
*Server binds to `http://127.0.0.1:8000` with WebSocket at `ws://127.0.0.1:8000/ws`.*

### Terminal 2 — Frontend (React + Vite + Tailwind)
```bash
cd frontend
npm run dev
```
*Frontend launches at `http://localhost:5173`.*

### Terminal 3 (Optional) — Automated Verification Suite
```bash
cd backend
python test_integration.py
```

---

## 3. Architecture & Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             REACT FRONTEND                                  │
│   VoiceButton (STT) ─── WebSocket Event Dispatcher ─── Results / Timeline   │
└──────────────┬──────────────────────────────────────────────▲───────────────┘
               │ ws://127.0.0.1:8000/ws                       │ WebSocket Events
               ▼                                              │
┌─────────────────────────────────────────────────────────────┴───────────────┐
│                        FASTAPI / TASK MANAGER (CORE)                        │
│                                                                             │
│  1. Parse Utterance ──► agent.py (Gemini / Mock Rule Extractor)             │
│  2. Interruption Check ──► Invalidate old task_id                           │
│  3. Browser Dispatch ──► browser/search.py (Playwright Headless/Headed)     │
│  4. Versioning Gate ──► is_current(task_id) [DISCARD IF SUPERSEDED]         │
│  5. Voice Synthesis ──► voice/rime.py (Rime TTS API)                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Interruption Lifecycle
1. User provides new speech or typed instruction during active browsing or speaking.
2. `TaskManager` registers new `task_id` and executes `_interrupt(old_task_id, new_task_id)`.
3. `voice_manager.interrupt_current_task()` invalidates active speech synthesis immediately.
4. `browser.cancel_task(old_task_id)` closes or cancels the obsolete Playwright context.
5. New search launches; any residual result from `old_task_id` is rejected by `is_current(task_id)` and silently discarded.

---

## 4. Exact Rime Configuration Specification

As required for submission verification, the exact Text-to-Speech specifications configured in `backend/voice/rime.py` and `backend/.env.example` are:

| Parameter | Configured Value | Environment Variable | Description |
|---|---|---|---|
| **Model ID** | `v1` (or `mist`) | `RIME_MODEL_ID` | Rime neural speech model identifier |
| **Speaker** | `marsh` | `RIME_SPEAKER` | Natural conversational English voice persona |
| **Language** | `en` (English) | Default | Standard English pronunciation |
| **Endpoint** | `https://users.rime.ai/v1/rime-tts` | `RIME_API_URL` | Official Rime TTS REST generation endpoint |
| **Audio Format** | `mp3` (22,050 Hz) | `RIME_AUDIO_FORMAT` | High-fidelity audio stream format |
| **Transport Used** | `HTTP POST (REST)` + `WebSocket Event Stream` | Internal pipeline | HTTP POST to Rime API; audio event streamed via WebSocket to client |

### Rime API Request Schema
```http
POST https://users.rime.ai/v1/rime-tts
Authorization: Bearer <RIME_API_KEY>
Content-Type: application/json
Accept: audio/mp3

{
  "speaker": "marsh",
  "text": "I found 3 matching options. The top pick is Christian Dior Sauvage for 14,500.",
  "modelId": "v1",
  "samplingRate": 22050,
  "speedAlpha": 1.0,
  "audioFormat": "mp3"
}
```

### Voice Interruption / Cancellation Mechanism
- If a task is interrupted while Rime is synthesizing or returning audio, `voice_manager.is_task_valid(task_id)` returns `False`, instantly dropping the response payload before it reaches the playback pipeline.
- If the `RIME_API_KEY` is not supplied or network fails, an internal audio synthesizer fallback is triggered to maintain test suite execution without pipeline halts.

---

## 5. Third-Party Services Used

1. **Rime TTS (`https://users.rime.ai`)**:
   - High-speed neural text-to-speech engine for conversational audio acknowledgements and search summaries.
2. **Google Gemini API (`gemini-2.5-flash` / `gemini-3.6-flash`)**:
   - Structured intent parsing, query constraint extraction (price caps, size, color), and contextual slot-filling.
3. **Microsoft Playwright (`Chromium`)**:
   - Automated real-browser navigation, anti-detection page evaluation, and e-commerce product extraction.
4. **Web Speech API / Local Whisper**:
   - Client-side and server-side speech-to-text transcription.

---

## 6. Known Limitations

- **E-Commerce Anti-Bot / Rate Limits**: Live e-commerce websites (Amazon, Flipkart) may periodically serve CAPTCHAs, bot walls, or dynamic layout variations.
- **Microphone Permissions**: Browser security policies require localhost or HTTPS origins for Web Speech API and MediaRecorder microphone access.
- **Single-User Task Versioning**: Central `TaskManager` is scoped for single active operator hackathon scenarios; distributed multi-user deployments require session-keyed task managers.

---

## 7. Failure Behavior & Graceful Fallbacks

| Failure Scenario | Built-in Mitigation / Fallback Behavior |
|---|---|
| **Amazon blocks automation or times out** | Automatically switches to secondary target (Flipkart). |
| **All live targets unreachable / blocked** | Seamlessly pivots to local fallback product catalogue (`fallback_page.html`) so judges and users always observe end-to-end extraction. |
| **Rime API key missing / Network error** | Falls back to internal mock audio tone generator to prevent pipeline crashes. |
| **Gemini API rate limit or outage** | Transparently drops back to zero-cost regex/rule-based parser (`parse_utterance_mock`). |
| **User interrupts mid-action** | Previous task state is invalidated; pending network results are discarded without UI update or audio overlap. |

---

## 8. Configuration Hygiene & Environment Variables

All secrets and environment configurations are managed via `backend/.env` with strict placeholder defaults in `backend/.env.example`:

```ini
# ==== Voice Layer ====
RIME_API_KEY=your_rime_api_key_here
RIME_SPEAKER=marsh
RIME_MODEL_ID=v1
RIME_API_URL=https://users.rime.ai/v1/rime-tts
RIME_AUDIO_FORMAT=mp3

# Speech-to-Text: "speech_recognition", "whisper_local", "gemini_api", or "mock"
STT_ENGINE=whisper_local
GEMINI_API_KEY=your_gemini_api_key_here

# ==== Browser Layer ====
HEADLESS=false
MAX_SEARCH_RESULTS=10

# ==== LLM Orchestration ====
LLM_ENGINE=mock
LLM_MODEL=gemini-2.5-flash

# ==== Server ====
HOST=127.0.0.1
PORT=8000
```

> [!NOTE]
> No production API keys, credentials, or `.env` files are tracked in version control. All repository branches pass preflight secret hygiene checks.

---

## 9. Automated Test Checklist

The integration test suite (`backend/test_integration.py`) validates the 6 core specification requirements:

```bash
cd backend
python test_integration.py
```

- **Test 1: Normal voice search** — Verifies full `task.started` → `task.completed` flow.
- **Test 2: Interrupt while AI is speaking** — Verifies old task cancellation and new task priority.
- **Test 3: Interrupt while browser is searching** — Verifies mid-browse abort and stale discard.
- **Test 4: Stale result arrival after new task** — Verifies `is_current()` rejects late-arriving results.
- **Test 5: Clean completion without interrupt** — Ensures no spurious interrupt events are broadcast.
- **Test 6: Simulated failure handling** — Confirms non-crashing graceful error output.
