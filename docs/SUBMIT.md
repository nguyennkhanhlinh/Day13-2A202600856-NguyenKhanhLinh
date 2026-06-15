# Submitting

The agent runs on a **real LLM** — set a key or a local endpoint first:
```bash
export OPENAI_API_KEY=sk-...                 # cloud (default model gpt-5.4-nano)
# OR free local: run Ollama/llama.cpp and set provider:"local" in config.json + LOCAL_BASE_URL
```

1. **Self-check** your scaffold (pure stdlib, no key):
   ```bash
   python harness/selfcheck.py
   ```
2. **Run the simulator** for the current phase:
   ```bash
   ./bin/<phase>/observathon-sim --config solution/config.json --wrapper solution/wrapper.py \
       --out run_output.json --concurrency 8
   #   <phase> = practice | public | private
   #   macOS first run: xattr -dr com.apple.quarantine bin/<phase>/*
   ```
3. **Score** the run (public/private phases):
   ```bash
   ./bin/<phase>/observathon-score --run run_output.json --findings solution/findings.json \
       --team <TEAM> --out score.json
   ```
4. **Commit & push** your `solution/` (config.json, prompt.txt, examples.json, wrapper.py,
   findings.json, your logs/traces) + `run_output.json` + `score.json`:
   ```bash
   git add solution/ run_output.json score.json && git commit -m "<team> <phase>" && git push
   ```

## Binaries by OS (`bin/<phase>/`)
| OS / arch | file |
|---|---|
| macOS (Apple Silicon, M1+) | `observathon-sim` / `observathon-score` (arm64) |
| Windows | unzip the folder, run `observathon-sim\observathon-sim.exe` / `observathon-score\observathon-score.exe` (keep the folder intact) |
| Linux | `observathon-sim` / `observathon-score` (x86_64) |

On **Windows** the binaries ship as a **folder** (PyInstaller onedir, to avoid a Python-DLL load error): unzip it and run `observathon-sim\observathon-sim.exe` from *inside* the folder — don't move the .exe out. (macOS Intel is not pre-built — Apple-Silicon, Windows and Linux are provided; on Intel,
run from source with Python + `openai`.)

## Phases
practice @ T0 → public **sim** @ T+1h, **score** @ T+2h → private **sim** @ T+3h, **score** @ T+3.5h.
The **private** phase adds a held-out, paraphrased set + the **injection** twist. Push your
private result once.
