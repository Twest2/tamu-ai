# TAMU AI — model roster and selection

Verified **2026-07-23**. The roster changes; always confirm with
`list_models()` or `python3 scripts/tamu_ai.py --list-models` rather than trusting
this file.

## Two gotchas about model ids

1. Every id carries a **`protected.` prefix** — `protected.gpt-5.4-mini`, not
   `gpt-5.4-mini`.
2. Several ids **contain spaces and capital letters** —
   `"protected.Claude Sonnet 4.6"`. They must be quoted, and they are not
   normalized: `protected.claude-sonnet-4.6` is a 403.

Note the inconsistency in the Claude family — most use spaces
(`protected.Claude Opus 4.8`) but Haiku 4.5 uses hyphens
(`protected.Claude-Haiku-4.5`). Copy ids from `list_models()`; don't guess.

## Available models

**Claude** (Bedrock; extended thinking always on — see the parameter constraints
in [quirks.md](quirks.md))

```
protected.Claude Opus 4.8      protected.Claude Sonnet 4.6
protected.Claude Opus 4.7      protected.Claude Sonnet 4.5
protected.Claude Opus 4.6      protected.Claude Sonnet 4
protected.Claude Opus 4.5      protected.Claude-Haiku-4.5
protected.Claude Opus 4.1      protected.Claude 3.5 Haiku
```

**GPT**

```
protected.gpt-5.5     protected.gpt-5.4        protected.gpt-5.2   protected.gpt-5.1
protected.gpt-5       protected.gpt-5.4-mini   protected.gpt-5-mini
protected.gpt-5.4-nano  protected.gpt-5-nano
protected.gpt-4.1     protected.gpt-4.1-mini   protected.gpt-4.1-nano  protected.gpt-4o
```

**Reasoning (o-series)**

```
protected.o3    protected.o3-mini    protected.o4-mini
```

**Gemini**

```
protected.gemini-3.5-flash   protected.gemini-3.1-flash-lite
protected.gemini-2.5-pro     protected.gemini-2.5-flash   protected.gemini-2.5-flash-lite
```

**Other**

```
protected.llama3.2
protected.text-embedding-3-small   ← LISTED BUT UNUSABLE, the endpoint is 403
```

## Choosing

| Job shape | Reach for | Why |
|---|---|---|
| Bulk mechanical work — classify, extract, tag thousands of rows | `protected.gpt-5.4-mini`, `protected.gemini-3.5-flash`, `protected.Claude-Haiku-4.5` | ~1 s/call, accept all standard parameters |
| Judgment — grading to a rubric, nuanced classification, anything where consistency with a human matters | `protected.Claude Sonnet 4.6`, `protected.Claude Opus 4.8` | Best rubric adherence measured here; Sonnet 4.6 hit r = 0.86–0.95 vs. a human grader across 457 answers |
| Hard reasoning, math, multi-step planning | `protected.gpt-5.5`, `protected.o3` | Built for it |
| **Guaranteed structured JSON** | any **GPT** model | `response_format` works there and is silently broken on Claude |
| Long context dumps | `protected.gemini-2.5-pro`, Claude models | Large windows |

Measured latency on short calls: `protected.gpt-5.4-mini` ≈ 1.0 s,
`protected.Claude Sonnet 4.6` ≈ 1.6–2.5 s.

## Cost

No pricing or quota information is exposed through the API, and the docs are
403-gated. Access is free to TAMU affiliates with a NetID, but treat quota as
unknown rather than unlimited: cache results so a re-run is free, and test a
prompt on 8–10 items before launching hundreds.

## A note on picking by name

Model quality does not track version numbers across families, and the roster gets
new entries regularly. When a job matters, run 8–10 items through two candidates
and compare the actual output — the quiz-grading work did exactly this
(Sonnet 4.6 vs. gpt-5.4-mini) before committing to a 457-call run.
