# tamu-ai

A [Claude Code](https://claude.com/claude-code) skill for using **Texas A&M's AI
gateway** (`chat-api.tamu.ai`) as an LLM backend — for one-off calls and for large
parallel jobs like grading, classifying, or extracting across thousands of rows.

Free to TAMU affiliates with a NetID. Fronts Claude, GPT, Gemini, o-series and
Llama models behind an OpenAI-compatible API.

## Why this exists

The gateway looks like the OpenAI API but diverges in ways that are easy to get
wrong and slow to diagnose. The official docs at `docs.tamus.ai` are 403-gated to
non-browser clients, so everything here was established by probing the live API:

- **Errors arrive in three different shapes.** The most common is `HTTP 200` with
  an `{"error": ...}` body — a status-code check passes, and your code dies later
  on a bare `KeyError: 'choices'`.
- **`response_format: json_object` is silently broken on Claude models.** It
  returns a normal `200` whose content is `{}`. Not an error — a wrong answer.
- **Claude models reject `temperature`, `top_p`, and `max_tokens`** outright,
  because extended thinking is permanently enabled on them.
- **`/embeddings` returns 403** even though an embedding model is advertised in
  the model list. There is no embeddings capability here.
- **JSON output is inconsistently fenced** — the same prompt returned
  ` ```json `-wrapped output on one call and bare JSON on the next.

The skill encodes all of this so you don't rediscover it.

## Install

```bash
git clone git@github.com:aedessler/tamu-ai.git ~/.claude/skills/tamu-ai
```

Requires Python 3.9+ and `requests` (`pip3 install requests`).

That's it — Claude Code picks up skills in `~/.claude/skills/` automatically.
Then just say **"use TAMU AI"** in any session.

To install for a single project instead, clone into `.claude/skills/tamu-ai/`
inside that project.

## API key

**No key is stored in this repo.** The client resolves one from, in order:

1. `$TAMU_AI_API_KEY`
2. `~/.config/tamu-ai/api-key`
3. Otherwise it fails with setup instructions — and Claude will ask you for one.

Get a key from [chat.tamu.ai](https://chat.tamu.ai) → Settings → API Key. To set
it up once so future sessions don't ask:

```bash
mkdir -p ~/.config/tamu-ai
printf %s 'sk-YOUR-KEY' > ~/.config/tamu-ai/api-key
chmod 600 ~/.config/tamu-ai/api-key
```

Or per-shell: `export TAMU_AI_API_KEY='sk-YOUR-KEY'`

Pass the key as an environment variable, never as a command-line flag — argv is
readable by other processes via `ps`.

## Verify

```bash
python3 ~/.claude/skills/tamu-ai/scripts/tamu_ai.py --check
```

```
Key OK. 33 models available.
Round-trip on protected.gpt-5.4-mini:
  ok
```

## Use it directly

The client works standalone, without Claude Code:

```bash
python3 scripts/tamu_ai.py --list-models
python3 scripts/tamu_ai.py --chat "explain lapse rate in one sentence"
python3 scripts/tamu_ai.py --chat "hello" -m "protected.Claude Sonnet 4.6"
```

```python
import os, sys
sys.path.insert(0, os.path.expanduser("~/.claude/skills/tamu-ai/scripts"))
from tamu_ai import chat, run_batch, extract_json, FAST_MODEL

chat("summarize this in one line", model=FAST_MODEL)

# Bulk: parallel, retried, cached, resumable
results, failures = run_batch(
    rows,
    build_messages=lambda r: f"Classify this: {r}",
    model=FAST_MODEL,
    workers=8,
    cache_path=".cache.json",   # re-runs are free for finished items
    parse=extract_json,         # optional
)
```

`run_batch` **returns** failures rather than raising, so one bad row can't discard
a long run, and the disk cache makes an interrupted job cheap to restart. Measured
at ~2.7–3.4 requests/sec with 8–10 workers.

## Contents

| Path | What |
|---|---|
| `SKILL.md` | Trigger rules, key handling, model selection, the traps |
| `scripts/tamu_ai.py` | Client + parallel batch runner. Importable module and CLI. |
| `references/quirks.md` | Full probe evidence with real response bodies |
| `references/models.md` | Model roster and selection guidance |

## Caveats

Model ids carry a `protected.` prefix and are not normalized — several contain
spaces and capitals (`"protected.Claude Sonnet 4.6"`), and the Claude family is
internally inconsistent (`protected.Claude-Haiku-4.5` uses hyphens). Copy ids from
`--list-models` rather than guessing; a wrong id returns `403 Model not found`.

The roster and the gateway's behavior both change. `references/` records what was
true on **2026-07-23**; re-probe if something looks off.

## License

MIT
