# Butterfly

**A self-correcting constraint architecture for adaptive AI agents.**

Butterfly sits between you and a language model and keeps the agent's behavior stable over a long conversation, the way a thermostat keeps a room steady. It doesn't retrain the model or touch its weights. It reads each reply, scores it, and adjusts two pieces of state that shape the next reply: a set of learned behavior rules and a personality profile.

A Python engine, a FastAPI server, and a React UI that shows the loop working in real time.

## The three methods

| | Method | What it does |
|---|---|---|
| 1 | **Positive Reinforcement Constraint System** | Rules have a strength. Good replies strengthen the active rules (+0.1), weak replies weaken them (−0.05). Rules that fall to 0.2 are pruned. The 2:1 asymmetry means one bad reply never kills a good rule. |
| 2 | **Mirror Loop with Personality Drift Correction** | Every 3 messages the agent reviews its recent replies, nudges its traits from what it observed, then pulls every trait 15% toward the group mean so no single trait runs away. |
| 3 | **Quality-Gated Constraint Extraction** | The inversion. A reply scoring 0.65 or higher teaches the agent *nothing new*; it only reinforces what's already working. New rules are learned only from replies that fell short, by reading the agent's own critique of that reply. |

All three share one quality score: the average of four checks (asked for clarification when the request was vague, length fit the request, followed active rules, admitted uncertainty when it should).

## Quick start (Windows)

You need **Python 3.10+**, **Node.js 18+**, and **Ollama**.

```powershell
# 1. Get a model (one time)
ollama pull llama3.1

# 2. Set up and run (one time setup is automatic)
.\start.ps1
```

`start.ps1` creates a virtual environment, installs everything, builds the UI on first run, and opens Butterfly in your browser at http://127.0.0.1:8000.

Options:

```powershell
.\start.ps1 -Model qwen2.5:14b    # use a different Ollama model
.\start.ps1 -Demo                 # no model needed; a built-in mock answers
.\start.ps1 -Judge                # the model grades its own replies instead of the heuristics
.\start.ps1 -SetupVoice           # add natural local voices (one time, ~350 MB)
```

If PowerShell refuses to run the script, allow local scripts once with
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

### Manual setup (any OS)

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate    macOS/Linux: source .venv/bin/activate
pip install -e ".[dev]"
cd frontend && npm install && npm run build && cd ..
python -m butterfly                  # or: python -m butterfly --demo
```

For frontend development with hot reload, run `python -m butterfly --no-browser` in one terminal and `npm run dev` in `frontend/` in another, then open http://localhost:5173.

## Using a different model backend

Butterfly is model-agnostic. Pick a backend with environment variables:

| Variable | Default | Meaning |
|---|---|---|
| `BUTTERFLY_BACKEND` | `ollama` | `ollama`, `openai` (any OpenAI-compatible server), or `mock` |
| `BUTTERFLY_MODEL` | `llama3.1` | model name |
| `OLLAMA_URL` | `http://localhost:11434` | point at Ollama on another machine, e.g. your desktop from a laptop |
| `OPENAI_BASE_URL` | | e.g. `http://localhost:1234/v1` for LM Studio |
| `OPENAI_API_KEY` | | if the server needs one |
| `BUTTERFLY_DATA` | `./data` | where saved state lives |

## Voices

**Read aloud** uses the browser's built-in voice until you install Kokoro, a free voice model that runs offline on your PC:

```powershell
.\start.ps1 -SetupVoice
```

That installs it and downloads the model into `models/kokoro/` (ignored by git). After that, a voice menu appears next to Read aloud with 50+ voices; picking one plays a short sample, and each persona remembers its own choice. Persona defaults are set with a `voice` entry, for example `"voice": {"name": "af_nova", "speed": 1.0}`. Nova uses `af_nova` and Echo uses `am_michael`. List every voice with `python -m butterfly.tts voices`.

Long replies are read in chunks, so the first sentence starts quickly while the rest is generated. If Kokoro isn't available, it falls back to the browser voice.

## Using it as a library

```python
from butterfly import ButterflyAgent, OllamaClient, ButterflyConfig

agent = ButterflyAgent(OllamaClient("llama3.1"), ButterflyConfig())
result = agent.turn("Can you help me fix this?")

print(result.response)
print(result.signals.quality_score, result.reinforced, result.rules_added)
print([c.text for c in agent.constraints])
print(agent.traits.traits)
```

Every parameter from the paper lives in `ButterflyConfig` with the paper's default values.

## Personas

Two personas from the original build ship by default: **Nova** (warm, curious) and **Echo** (precise, economical). Each keeps its own rules, personality, and history, saved in `data/`. Add your own by creating `data/personas.json`:

```json
{
  "sage": {
    "name": "Sage",
    "description": "You are calm and patient.",
    "traits": {"curious": 0.5, "formal": 0.4, "verbose": 0.4},
    "seed_rules": ["Avoid slang in replies."],
    "voice": {"name": "bf_emma", "speed": 1.0}
  }
}
```

## Project layout

```
butterfly/
  config.py        every reference parameter (paper Section 11.2)
  constraints.py   Method 1: rules, strengthen/weaken, pruning, dedupe
  traits.py        Method 2 state: trait profile and drift correction
  quality.py       the shared quality score (heuristic or model-graded)
  extractor.py     Method 3: the quality gate and rule extraction
  agent.py         the four-stage loop that ties it all together
  prompts.py       system, self-evaluation, and mirror-loop prompts
  llm.py           Ollama, OpenAI-compatible, and mock backends
  server.py        FastAPI API and UI hosting
  personas.py      Nova, Echo, and custom personas
  store.py         saves state to JSON between runs
  tts.py           Kokoro text-to-speech (optional)
frontend/          React + TypeScript UI
tests/             63 tests: unit, integration, simulations, API, voice
docs/              white paper and implementation notes
```

## Tests

```bash
pytest
```

The suite checks the paper's claims directly: the 0.65 gate, the 2:1 asymmetry (a rule alternating good and bad turns rises from 1.0 to 1.5 over 20 turns), pruning after exactly 16 straight failures, the mirror loop firing at messages 3, 6, and 9, drift correction preserving the trait mean and ordering, and long-run simulations for stability.

See [`docs/IMPLEMENTATION_NOTES.md`](docs/IMPLEMENTATION_NOTES.md) for where this implementation fills gaps the paper leaves open, and the known limitations.

## License

Apache License 2.0. See [`LICENSE`](LICENSE). The architecture is described in the white paper by RJ Thomas, released openly without patent protection. Please keep the author's name and the paper's publication date in derivative works.
