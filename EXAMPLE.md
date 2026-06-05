# junior-dev — try it on another machine

A hands-on walkthrough: set up the prerequisites, install the plugin, run the
tests, and watch Claude delegate a real task to a local model. Copy-paste each
block; expected output is shown so you can tell if a step worked.

---

## 0. Prerequisites

You need three things on the machine:

- **[Ollama](https://ollama.com)** — the local model server
- **A pulled model** — this guide uses `gemma4:12b`
- **Python 3** — standard library only (no `pip install`, no `curl` needed)

```bash
# Install Ollama (Linux/macOS); see ollama.com for other platforms
curl -fsSL https://ollama.com/install.sh | sh

# Start the server (leave it running, or use your OS service)
ollama serve &

# Pull the model (~7.6 GB)
ollama pull gemma4:12b

# Sanity check: the API should list your model
curl -s http://localhost:11434/api/tags
```

Expected: a JSON blob listing `gemma4:12b`.

---

## 1. Install the plugin in Claude Code

Inside Claude Code, add the marketplace and install:

```
/plugin marketplace add davidnet/junior-dev
/plugin install junior-dev@junior-dev
```

> Testing a local checkout instead of GitHub? Point the marketplace at the
> folder: `/plugin marketplace add /path/to/junior-dev`

Then **reload Claude Code** so the skill and scripts load.

Verify it registered:

```
/plugin
```

Expected: `junior-dev` shows as installed/enabled, and the skill
`/junior-dev:junior` is available.

---

## 2. Configure (optional)

By default the plugin auto-detects the first installed model and talks to
`http://localhost:11434`. Override via env vars in your shell profile:

```bash
export JUNIOR_MODEL=gemma4:12b             # pin a specific model
export OLLAMA_HOST=http://localhost:11434  # or a remote Ollama box
export JUNIOR_THINK=0                       # 1 = always let the model reason first
```

---

## 3. Check the local model is reachable

The plugin ships a preflight script. From a Claude Code Bash call (or a normal
terminal with the plugin's `bin/` on `PATH`):

```bash
junior-preflight.py
```

Expected:

```
junior-dev: ready. Host=http://localhost:11434
  Installed: gemma4:12b
  Using:     gemma4:12b  (auto: first model)
```

If instead you see "Ollama NOT reachable", start `ollama serve` or fix
`OLLAMA_HOST`.

---

## 4. Run the automated tests

From the cloned repo:

```bash
python3 -m unittest discover -s tests   # logic tests, no network
python3 tests/smoke.py                  # live round-trip vs Ollama
```

Expected: `OK` (10 tests) for the first, and for the smoke test:

```
preflight OK — models: gemma4:12b; using: gemma4:12b
--- generated ---
def add(a, b):
    return a + b
-----------------
PASS: generation produced correct, clean, runnable code.
```

(The smoke test prints `SKIP: ...` instead if Ollama is down — that's fine.)

---

## 5. Watch the skill delegate a real task

This is the part that proves the whole loop. In a **new** Claude Code
conversation, give it a clearly junior task **without** naming the skill:

> write a Python function `roman_to_int(s)` that converts a Roman numeral to an integer

What you should observe Claude do:

1. **Triage** — recognize it as a self-contained, verifiable, junior task
2. **Preflight** — confirm the local model is up
3. **Confirm** — tell you what it will delegate and how it will verify, and ask for a quick OK
4. **Delegate** — call `junior-delegate.py` (turning on thinking because there are edge cases)
5. **Verify** — run the generated code against known values (`IV=4`, `MCMXCIV=1994`, …)
6. **Report** — integrate it and tell you it came from the local model

You can also invoke it explicitly:

```
/junior-dev:junior write unit-test stubs for a module called utils.py
```

---

## 6. Confirm it does NOT over-delegate

Give it a task that needs judgment and broad context:

> refactor the auth module to fix the session race condition

Expected: Claude does **not** delegate — it handles this itself. Delegation is
deliberately conservative; junior tasks only.

---

## 7. Try the manual pipeline directly (no Claude needed)

You can drive the delegate script yourself to feel the raw round-trip:

```bash
printf '%s' "Write a Python function is_even(n) returning a bool. Return ONLY code." \
  | junior-delegate.py

# Let the model reason first (free locally; better first pass on logic):
printf '%s' "Write a Python function that returns the nth Fibonacci number iteratively. Return ONLY code." \
  | JUNIOR_THINK=1 junior-delegate.py
```

Expected: clean code on stdout — no markdown fences, no reasoning text.
`gemma4:12b` routes its reasoning to a separate `thinking` field, so
`JUNIOR_THINK=1` does not leak a thought block into the output.

---

## 8. Test the failure path

Stop Ollama and confirm graceful degradation:

```bash
ollama stop gemma4:12b 2>/dev/null; pkill -f "ollama serve" 2>/dev/null

junior-preflight.py   # should report NOT reachable, exit non-zero
```

In Claude Code, a junior task should now make Claude notice the local model is
down (via preflight) and **just do the task itself** instead of erroring.

Restart with `ollama serve &` when done.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Ollama NOT reachable` | `ollama serve` is not running, or `OLLAMA_HOST` is wrong |
| `no model available` | `ollama pull gemma4:12b`, or set `JUNIOR_MODEL` |
| `junior-delegate.py: command not found` | Reload Claude Code so the plugin's `bin/` is on `PATH`; or call it by full path |
| Permission prompts every call | The bundled allowlist may need confirming once; approve it |
| Output has ` ``` ` fences or reasoning text | File an issue — `clean()` in `lib/junior_ollama.py` should strip these |
