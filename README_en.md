# Observathon — Student Kit

[🇻🇳 Tiếng Việt](README.md) | 🇬🇧 English

You have an **opaque, silent, buggy** e-commerce agent (a binary) that runs on a **real LLM**.
It tells you nothing. Your job: **instrument it, diagnose the faults, and fix them** — by
correcting the config, **rewriting the agent's system prompt**, and adding a mitigation wrapper.

## Setup (a real LLM is required)
```bash
# 1. pick an engine:
export OPENAI_API_KEY=sk-...        # cloud (default model gpt-5.4-nano), OR
#    free local: run Ollama / llama.cpp (OpenAI-compatible) and set provider:"local" + LOCAL_BASE_URL in config.json

# 2. confirm your submission scaffold is well-formed (pure stdlib, no key)
python harness/selfcheck.py

# 3. run the PRACTICE simulator binary (in bin/practice/)
./bin/practice/observathon-sim --config solution/config.json --wrapper solution/wrapper.py \
    --out run_output.json --concurrency 8
#   macOS first run: xattr -dr com.apple.quarantine bin/practice/observathon-sim
#   Windows:        bin\practice\observathon-sim\observathon-sim.exe ...   (exe lives INSIDE its folder)
```
The agent emits **nothing** and `run_output.json` is **lean by design** — every row has only
`answer` + `status` (no latency, no tokens, no tool calls, no trace). The ONLY way to see
latency, cost, tool usage, loops, drift and PII is to **instrument `solution/wrapper.py`**:
`call_next()` returns the full result (incl. `meta` + `trace`) to *you* — log it with the
`telemetry/` toolkit from Day 13. (The sim also writes a signed `sealed` block for the scorer's
tamper-proof metrics — that is not your observability.)

## What you optimize (the v6 lever)
The agent is **prompt-driven** and ships with a deliberately **bad** system prompt (it
fabricates totals, miscomputes, over-calls tools, echoes PII, and obeys instructions hidden in
order notes). **Rewrite `solution/prompt.txt`** — it's the highest-leverage fix and a visible
**15% `prompt` score**. See **[`docs/PROMPT_OPTIMIZATION.md`](docs/PROMPT_OPTIMIZATION.md)**.

| You edit | What it does |
|---|---|
| `solution/config.json` | agent knobs (provider/model, temperature, retry, cache, normalize, redact, `self_consistency`, `tool_budget`, `planner`, …) |
| `solution/prompt.txt` | the agent **system prompt** — rewrite it |
| `solution/examples.json` | optional few-shot |
| `solution/wrapper.py` | `mitigate()` — observability + retry/cache/route/redact/sanitize + per-request prompt routing |
| `solution/findings.json` | your diagnosis (fault class + evidence + root cause) |

## Which binary for your OS (`bin/<phase>/`)
| OS / arch | file |
|---|---|
| macOS (Apple Silicon, M1+) | `observathon-sim` / `observathon-score` (arm64) |
| Windows | unzip the folder, run `observathon-sim\observathon-sim.exe` / `observathon-score\observathon-score.exe` (keep the folder intact) |
| Linux | `observathon-sim` / `observathon-score` (x86_64) |

(macOS Intel isn't pre-built — on Intel, run from source with Python + `openai`.) macOS
Gatekeeper first run: `xattr -dr com.apple.quarantine bin/<phase>/*`. Release schedule:
`practice` from the start · public **sim** @ 1h, **score** @ 2h · private **sim** @ 3h, **score** @ 3.5h.

## Generate realistic traffic (choose your load)
```bash
# 200 active users x 12 turns each = 2400 requests over a simulated time window
./bin/practice/observathon-sim --users 200 --turns 12 --concurrency 12 \
    --config solution/config.json --wrapper solution/wrapper.py --out run_output.json
```
- `--users N` active users · `--turns K` requests per user (higher K → more quality-drift over time) · `--rps` arrival rate · `--concurrency` parallel requests.
- **Practice traffic is RANDOM every run** (it prints `random run seed = …`; pass `--seed <that>` to reproduce). Scoring always uses the **fixed** public/private set, so every team is ranked on identical traffic.

## Scoring
`100 × (0.32·correct + 0.16·quality + 0.13·error + 0.08·latency + 0.09·cost + 0.07·drift +
0.15·prompt) + up to 22 × diagnosis-F1`. Quality = LLM judge (`gpt-5.4-mini`, offline fallback).
`prompt` is outcome-based (grounding/arithmetic/tool-economy/PII/injection minus bloat).

## What you submit (git push `solution/` + `run_output.json` + `score.json`)
`config.json` · `prompt.txt` · `examples.json` (optional) · `wrapper.py` · `findings.json`.

## Phases
- **Now → 1h**: diagnose with the practice binary; rewrite the prompt + config.
- **1h** public **sim** · **2h** public **score** → commit, push, climb.
- **3h** private **sim** (held-out + paraphrase + **injection** twist) · **3.5h** private **score** → push (final).

See `docs/FAULT_CLASSES.md`, `docs/PROMPT_OPTIMIZATION.md`, `docs/WRAPPER_API.md`, `docs/SUBMIT.md`. Rules: `../RULES.md`.
