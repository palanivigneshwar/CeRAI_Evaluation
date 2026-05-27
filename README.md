# CeRAI AIEvaluationTool — Evaluation of Gemma3n:e2b

Gates Foundation AI Fellowship India 2026 — Technical Assignment (Option A)

## What this repository contains

- **Bug fixes** to CeRAI AIEvaluationTool v1.2 required to run API-mode evaluation
- **Target configuration** for Ollama-hosted local models
- **Evaluation results** from a 10-case Dialogue_Coherence run against Gemma3n:e2b
- **Live report**: `index.html` — open in any browser

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.10+ | |
| Node.js | 20.19+ or 22.14+ | Must be on system PATH (not inside venv) |
| Ollama | 0.3.0+ | See note below |
| OS | Windows 10/11 or Linux | Mac untested |

> **Ollama version note**: The tool uses OpenAI-compatible `/v1` endpoints. Ollama 0.24.0 does not support this. Either upgrade to 0.3.0+ or apply the patch in [Bug #03](#bug-03) below.

---

## Setup

### 1. Clone and install

```bash
git clone https://github.com/cerai-iitm/AIEvaluationTool
cd AIEvaluationTool
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Apply required bug fixes

**Bug #01 — Hardcoded port** (`src/app/TestCaseExecutorDashboard/back-end/main.py`, line ~769):
```python
# BEFORE
client = InterfaceManagerClient(base_url="http://localhost:8000", ...)

# AFTER
interface_manager_port = port_config.get("interface-manager", "8001")
client = InterfaceManagerClient(base_url=f"http://localhost:{interface_manager_port}", ...)
```

**Bug #02 — Language filter** (`src/lib/orm/DB.py`, line ~1562):
```python
# BEFORE — crashes if lang_names is a string
.where(Languages.lang_name.in_(lang_names))

# AFTER
lang_list = [lang_names] if isinstance(lang_names, str) else lang_names
.where(Languages.lang_name.in_(lang_list))
```

**Bug #03 — Ollama native API** (`src/app/interface_manager/api_handler.py`):
```python
# Replace _run_local() with:
def _run_local(ctx, prompt: str) -> str:
    import requests
    response = requests.post(
        "http://localhost:11434/api/chat",
        json={
            "model": ctx.agent_name,
            "messages": [{"role": "user", "content": prompt}],
            "stream": False
        },
        timeout=120
    )
    response.raise_for_status()
    return response.json()["message"]["content"]
```

**Bug #04 — UTF-8 encoding** (`src/app/importer/main.py`):
```python
# Add encoding='utf-8' to all open() calls:
plans = json.load(open(config['files']['plans'], 'r', encoding='utf-8'))
prompts = json.load(open(config['files']['testcases'], 'r', encoding='utf-8'))
```

### 3. Start Ollama and pull model

```bash
ollama serve
ollama pull gemma3n:e2b  # or: ollama pull llama3.1:8b
```

### 4. Configure database and target

Edit `src/app/importer/config.json`:
```json
{
  "db": { "engine": "sqlite", "file": "AIEvaluationData.db" },
  "files": {
    "plans": "data/plans.json",
    "testcases": "data/DataPoints.json",
    "strategies": "data/strategy_id.json"
  }
}
```

Add your target at the end of `src/app/importer/main.py`:
```python
tgt = Target(
    target_name="gemma3n-e2b",
    target_type="API",
    target_url="http://localhost:11434",
    target_description="Local Gemma3n:e2b via Ollama",
    target_domain="Local API Interface",
    target_languages=["english"]
)
target_id = db.add_or_get_target(target=tgt)
```

Run from repo root:
```bash
python src/app/importer/main.py
```

### 5. Configure Interface Manager

Edit `src/app/interface_manager/config.json`:
```json
{
  "base_url": "http://localhost:8001",
  "application_type": "API",
  "agent_name": "gemma3n:e2b",
  "application_name": "gemma3n-e2b",
  "application_url": "http://localhost:11434",
  "ollama_server_url": "http://localhost:11434",
  "debug": "True",
  "headless": "False"
}
```

Set `.env` (copy from `.env.example`):
```env
OLLAMA_URL=http://localhost:11434
LLM_AS_JUDGE_MODEL=gemma3n:e2b
OPENAI_API_KEY=ollama
OPENAI_BASE_URL=http://localhost:11434/v1
```

### 6. Start all services (4 terminals)

```bash
# Terminal 1 — Interface Manager (port 8001)
cd src/app/interface_manager
python main.py

# Terminal 2 — TestCaseExecutorDashboard backend (port 7000)
cd src/app/TestCaseExecutorDashboard/back-end
python main.py

# Terminal 3 — TestCaseExecutorDashboard frontend (port 3000)
# Run WITHOUT venv activated, with Node in PATH
cd src/app/TestCaseExecutorDashboard/front-end
npm install && npm start

# Terminal 4 — TDMS backend (port 8000) [optional, for test data management]
cd src/app/TDMS/back-end
python main.py
```

### 7. Run evaluation

1. Open `http://localhost:3000`
2. Click **New Test Run**
3. Fill: Target=`gemma3n-e2b`, Test Plan=`Conversational_Quality`, Metric=`Dialogue_Coherence`, Domain=`agriculture`, Max Cases=`10`
4. Leave Language blank (see Bug #02)
5. Click **Start Run**

### 8. Run response analyzer

```bash
cd src/app/response_analyzer
python analyze.py --run-name "your-run-name"
```

### 9. View report

Download the Excel report from the dashboard, or open `index.html` in a browser.

---

## Bugs filed

| # | Title | Severity | File |
|---|-------|----------|------|
| 1 | Hardcoded port 8000 ignores config.json | Blocking | `TestCaseExecutorDashboard/back-end/main.py:769` |
| 2 | Language filter passes string not list to SQLAlchemy IN | Blocking | `lib/orm/DB.py:1562` |
| 3 | Ollama /v1 endpoint not supported in v0.24.0 | Blocking | `interface_manager/api_handler.py:135` |
| 4 | UTF-8 encoding missing on Windows for Indian language data | Windows-blocking | `importer/main.py:92,95,98` |
| 5 | SLOR metric incompatible with non-English responses | Design flaw | `lib/strategy/fluency_score.py` |

---

## AI use disclosure

Claude (Sonnet 4.6) was used throughout this assignment as a debugging partner:

- **Tracing errors**: Each traceback was shared with Claude to identify the root cause and suggest a minimal fix. This was faster than reading through the full codebase cold.
- **Configuration**: Claude helped interpret the multi-service architecture from the README and identify which config values controlled which services.
- **Report writing**: The live endpoint HTML and this README were drafted with Claude's assistance, then reviewed and edited for accuracy against the actual run results.
- **Where it went wrong**: Claude initially suggested the wrong Ollama endpoint (`/api/generate` instead of `/api/chat`) and assumed a newer Ollama version supported `/v1`. Both required verification against actual Ollama behaviour. Claude also suggested `ollama pull gemma3:2b` when the correct model name was `gemma3n:e2b` — model naming in Ollama's registry is not something Claude knows reliably.

The core debugging loop (read error → hypothesise fix → apply → verify) was done manually. Claude accelerated it; it did not replace it.
