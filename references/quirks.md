# TAMU AI gateway — observed behavior

Every entry below was established by probing the live API on **2026-07-23**. The
official documentation at `https://docs.tamus.ai/docs/prod/api-tool/` returns
**HTTP 403** to any non-browser client, so this file is the working record.

Base URL: `https://chat-api.tamu.ai/openai`
Auth: `Authorization: Bearer sk-...` (key from chat.tamu.ai → Settings → API Key)

The gateway is LiteLLM in front of Open WebUI, routing to Bedrock (Claude), Azure/
OpenAI (GPT), and Google (Gemini). Most of the surprises come from that layering:
errors from three different tiers reach you in three different formats.

---

## Endpoint inventory

| Endpoint | Method | Status |
|---|---|---|
| `/models` | GET | works — returns `{"data": [{"id": ...}]}` |
| `/chat/completions` | POST | works |
| `/embeddings` | POST | **403, blocked** |

The embeddings block is worth knowing about because the model list actively
misleads you — `protected.text-embedding-3-small` is listed as available:

```
POST /openai/embeddings  {"model": "protected.text-embedding-3-small", "input": "hello"}
→ HTTP 403
  {"detail":"Direct API passthrough is disabled. Set ENABLE_OPENAI_API_PASSTHROUGH=True to enable."}
```

Identical response with the bare model name (no `protected.` prefix). **There is
no embeddings capability on this gateway.** Anything needing vectors — semantic
search, clustering, dedup — has to use a local model (e.g. `sentence-transformers`)
or a different provider.

---

## The three error shapes

A `resp.raise_for_status()` / `resp.ok` check catches only one of these.

### 1. HTTP 200 with an `error` body — the trap

Upstream model errors come back with a **success status code**:

```
POST /chat/completions  {"model":"protected.Claude Sonnet 4.6", ..., "temperature":0}
→ HTTP 200
  {"error":{"message":"litellm.BadRequestError: BedrockException - {\"message\":\"The model
    returned the following errors: `temperature` may only be set to 1 when thinking is
    enabled...\"}","type":null,"param":null,"code":"400"}}
```

Note the `"code": "400"` *inside* a 200 response. Code that trusts the status code
proceeds to `payload["choices"][0]` and dies with a bare `KeyError: 'choices'`
several frames from the real cause.

**Always check for an `error` key before reading `choices`.**

### 2. HTTP 403 with a `detail` body — gateway rejection

```
POST /chat/completions  {"model":"nope", ...}
→ HTTP 403  {"detail":"Model not found"}
```

Note this is **403, not 404**, for an unknown model — and the body uses `detail`,
not `error`. Not retryable. Almost always a mistyped model id (they contain spaces
and a `protected.` prefix, so this is easy to hit).

### 3. Transport timeouts

Seen once on an otherwise ordinary request that succeeded on retry. Use a generous
timeout (180 s) and retry with backoff.

---

## Parameter support by model family

`protected.Claude *` models run on Bedrock **with extended thinking permanently
enabled**, which makes several ordinary parameters illegal. Verified individually:

| Parameter | `protected.Claude Sonnet 4.6` | `protected.gpt-5.4-mini` |
|---|---|---|
| *(none sent)* | OK | OK |
| `temperature: 1` | OK | OK |
| `temperature: 0` | error — "`temperature` may only be set to 1 when thinking is enabled" | OK |
| `max_tokens: 50` | error — "`max_tokens` must be greater than `thinking.budget_tokens`" | OK |
| `max_completion_tokens: 50` | same error | OK |
| `top_p: 0.9` | error — "`top_p` must be greater than or equal to 0.95 or unset" | OK |
| `response_format: {"type":"json_object"}` | **HTTP 200, content is `"{}"`** | OK, real JSON |

### `response_format` is the dangerous one

On Claude models it does not error. It returns a normal-looking 200 whose content
is the two characters `{}`. A pipeline that trusts it gets empty results that parse
fine as JSON and quietly poison whatever comes next.

**Working approach on Claude:** state the JSON contract in the prompt text and
parse defensively. This is what the quiz grader does and it produced 457/457 valid
responses:

```
Respond with ONLY a JSON object, no prose and no code fences:
{"score": <number 0-15>, "comment": "<max 12 words>"}
```

Practical rule: **send no sampling parameters to Claude models at all.**

---

## Output formatting is inconsistent — parse defensively

Asked for `{"ok":true}` with no `response_format`, across separate runs:

| Model | Observed content |
|---|---|
| `protected.Claude Sonnet 4.6` | ` ```json\n{"ok":true}\n``` ` **and**, on a later identical call, bare `{"ok": true}` |
| `protected.gpt-5.4-mini` | bare `{"ok":true}` every time |

Claude **sometimes** fences its JSON and sometimes does not — the same prompt
produced both. Never assume either form; a parser that only handles bare JSON will
work in testing and fail intermittently in production. Strip optional fences and
fall back to a regex scan for the outermost object — `extract_json()` in
`scripts/tamu_ai.py` does exactly this.

---

## Response metadata

The `usage` block is present and includes cache accounting:

```json
{"completion_tokens": 20, "prompt_tokens": 44, "total_tokens": 64,
 "prompt_tokens_details": {"cached_tokens": 0, "cache_creation_tokens": 0},
 "cache_creation_input_tokens": 0, "cache_read_input_tokens": 0}
```

---

## Throughput

From the 457-call quiz-grading run:

| Measure | Value |
|---|---|
| Sustained rate, 10 workers | 2.7–3.4 req/s |
| Full 457-call job | 175 s, 0 failures |
| `protected.Claude Sonnet 4.6` latency | ~1.6–2.5 s per short call |
| `protected.gpt-5.4-mini` latency | ~0.9–1.0 s per short call |
| Rate limiting observed | none at 10 concurrent |

8 workers is a safe starting point; 10 was tested clean. No documented quota was
discoverable, so don't assume unlimited — cache results so a re-run is free.
