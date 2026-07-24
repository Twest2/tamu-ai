---
name: tamu-ai
description: Use the Texas A&M University AI system (TAMU AI, chat.tamu.ai, chat-api.tamu.ai) as the LLM backend for a task. Trigger whenever the user says "use TAMU AI", "TAMU AI", "the TAMU API", "TAMU GPT/LLM", "Texas A&M AI", or names chat.tamu.ai / tamu.ai as where a request should go — and for all follow-on work in a session where TAMU AI is already the chosen backend. Covers single calls and large parallel jobs (grading, classifying, summarizing, extracting across many rows). The gateway is OpenAI-compatible on the surface but fails in three different shapes and silently breaks response_format on Claude models, so use this skill rather than writing a plain OpenAI snippet. This skill never contains an API key — it resolves one from the environment or a user-owned config file, and asks the user if neither exists.
---

# TAMU AI

Texas A&M's LLM gateway. OpenAI-compatible at
`https://chat-api.tamu.ai/openai`, fronting Claude, GPT, Gemini, o-series and
Llama models. Free to TAMU affiliates with a NetID.

**It is not a drop-in OpenAI endpoint.** It fails in three different shapes, and
one common parameter fails *silently* — returning empty content instead of an
error. Read "The traps" below before writing any call by hand.

## 1. Get the API key — never hardcode it

**The key must never be written into a script, a project file, this skill, or a
shell profile.** Resolve it in this order:

1. `$TAMU_AI_API_KEY` in the environment
2. `~/.config/tamu-ai/api-key`
3. **Ask the user for it**, then use it for the session

Check the first two before asking — the user may already have it set up:

```bash
[ -n "$TAMU_AI_API_KEY" ] && echo "env: set" || echo "env: unset"; [ -f ~/.config/tamu-ai/api-key ] && echo "file: present" || echo "file: absent"
```

If neither exists, ask the user: *"What's your TAMU AI API key? Get one from
chat.tamu.ai → Settings → API Key."*

Once you have a key that works, **offer** to save it so future sessions don't ask
again — and only write it after they say yes:

```bash
mkdir -p ~/.config/tamu-ai && printf %s 'sk-REPLACE' > ~/.config/tamu-ai/api-key && chmod 600 ~/.config/tamu-ai/api-key
```

Never save it unprompted. If the user declines, pass it inline per command
(`TAMU_AI_API_KEY="sk-..." python3 script.py`) and write nothing.

**Pass the key as an environment variable, never as a command-line flag** — argv
is visible to other processes via `ps`. Never echo it, log it, or include it in a
file you write.

## 2. Use the client

`scripts/tamu_ai.py` already encodes everything below. Prefer it over hand-rolled
requests.

```bash
python3 ~/.claude/skills/tamu-ai/scripts/tamu_ai.py --check          # verify key + round-trip
python3 ~/.claude/skills/tamu-ai/scripts/tamu_ai.py --list-models
python3 ~/.claude/skills/tamu-ai/scripts/tamu_ai.py --chat "hello" -m protected.gpt-5.4-mini
```

```python
import os, sys
sys.path.insert(0, os.path.expanduser("~/.claude/skills/tamu-ai/scripts"))
from tamu_ai import chat, list_models, run_batch, extract_json, FAST_MODEL

chat("summarize this in one line", model=FAST_MODEL)
```

(`sys.path` does not expand `~` on its own — `expanduser` is required. If the
skill lives elsewhere, point that path at wherever `tamu_ai.py` actually is.)

## 3. Pick a model deliberately

There is **no default** — the right choice depends on the job. Model ids carry a
`protected.` prefix and contain spaces (`"protected.Claude Sonnet 4.6"`).

| Job shape | Reach for |
|---|---|
| Bulk, mechanical, high volume | `protected.gpt-5.4-mini`, `protected.gemini-3.5-flash`, `protected.Claude-Haiku-4.5` |
| Judgment: grading, rubrics, nuanced classification | `protected.Claude Sonnet 4.6`, `protected.Claude Opus 4.8` |
| Hard reasoning, math, multi-step | `protected.gpt-5.5`, `protected.o3` |
| Structured JSON output | a **GPT** model — `response_format` works there and is broken on Claude |

The roster changes; confirm with `list_models()` rather than trusting this table.
Full list and tradeoffs: [references/models.md](references/models.md).

## The traps

Read these even if you are writing your own request code.

**Only two endpoints work.** `GET /models` and `POST /chat/completions`.
`POST /embeddings` returns 403 *"Direct API passthrough is disabled"* even though
`protected.text-embedding-3-small` appears in the model list. There is no
embeddings capability here — don't design around one.

**Errors arrive in three shapes. Checking the status code is not enough.**

1. `HTTP 200` with `{"error": {...}}` in the body — upstream model errors land
   here. Code that only checks `resp.ok` will sail straight into a `KeyError` on
   `["choices"]`.
2. `HTTP 403` with `{"detail": "Model not found"}` — gateway-level rejection. Not
   retryable; usually a typo'd model id.
3. Transport timeouts, occasionally on an otherwise fine request. Retry these.

**Claude models run with extended thinking permanently on**, so Bedrock rejects
several standard parameters:

| Parameter | On `protected.Claude *` | On `protected.gpt-*` |
|---|---|---|
| `temperature: 1` | fine | fine |
| `temperature: 0` | **error** — only 1 allowed when thinking is on | fine |
| `max_tokens`, `max_completion_tokens` | **error** — must exceed the thinking budget | fine |
| `top_p: 0.9` | **error** — must be ≥ 0.95 or unset | fine |
| `response_format: json_object` | **silently returns `{}`** | fine |

Simplest rule: **send no sampling parameters at all to Claude models.** The
client drops them with a warning; `response_format` it refuses outright, because
silently dropping it would leave you believing you had guaranteed JSON.

**To get JSON out of a Claude model, ask for it in the prompt** and parse with
`extract_json()`. Claude *sometimes* wraps JSON in ` ```json ` fences and
sometimes returns it bare — the same prompt produced both — so never assume one
form. `extract_json` handles either.

## Bulk jobs

For anything over ~20 calls use `run_batch`, which does parallelism, retry with
backoff, a disk cache, and resume:

```python
results, failures = run_batch(
    rows,                                   # any sequence
    build_messages=lambda r: f"Classify: {r['text']}",
    model=FAST_MODEL,
    workers=8,
    cache_path="/path/to/.cache.json",      # re-runs are free for finished items
    prompt_version=1,                       # bump to invalidate after prompt edits
    parse=extract_json,                     # optional
)
```

Two properties that matter on long runs:

- **Failures are returned, not raised.** One bad row cannot throw away 400
  finished ones. Always check `failures` and report them — never let a failed
  call silently become a zero or an empty string in your output.
- **The cache makes re-running cheap**, so an interrupted job just gets run
  again. Bump `prompt_version` when you change a prompt, or you will serve stale
  results.

~2.7–3.4 requests/sec at 8–10 workers, with no rate limiting observed. Start at 8.

**Shape of a real job this was built from:** grading 457 short answers from a
Canvas quiz export — one call per answer carrying that question's rubric, run at
8–10 workers, cached to disk, completed in 175 s with zero failures. The pattern
generalizes: one small single-purpose call per row beats one giant call over the
whole file, both for accuracy and because a failure then costs one row instead of
the batch.

## Reference

- [references/quirks.md](references/quirks.md) — full probe evidence with real
  response bodies. The official docs at `docs.tamus.ai` are 403-gated, so this is
  the record.
- [references/models.md](references/models.md) — roster and selection guidance.
