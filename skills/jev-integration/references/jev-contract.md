# Jev API and SDK contract

The factual contract: inputs, outputs, primitives, confidence, SDKs, limits. The
thinking behind when and how to use them is in `SKILL.md`. Verify current details
against the live docs: https://docs.typesafe.ai/llms.txt (API:
https://docs.typesafe.ai/api.md, primitives: https://docs.typesafe.ai/primitives.md).

## The call

One shape for everything: **state + questions in, answers out.**

```
POST https://api.typesafe.ai/v1/systemone
Authorization: Bearer $TYPESAFE_API_KEY
Content-Type: application/json

{
  "state": <string | object | array>,
  "model": "jev-latest",
  "questions": {
    "<id you choose>": {
      "type": "choice" | "score" | "noul",
      "instructions": <string | object | array>,
      "criteria": <depends on type>
    }
  }
}
```

Question ids are for your code only. They are never sent to the model.

## Input: state

What the questions are judged against. Pass it directly to the SDK.

| Shape | Use for | Example |
| - | - | - |
| string | one message, article, or passage | `"My card was charged twice."` |
| object | named fields, records, app state (preferred) | `{"ticket": {...}, "policy": "..."}` |
| array | sequences of messages or records | `["Hi", "Charged twice"]` |

- Text only. Pre-process images/audio/video/binaries to text or structured fields.
- Prefer named JSON fields. Reference nested values in questions with backticked
  dot-paths: `` "Does `ticket.messages[0].text` request a refund?" ``.
- Include only what the questions need. Unrelated detail reduces accuracy.
- Keep the same state together when a judgment compares its parts.

## Input: questions

| Field | Choice | Score | Noul |
| - | - | - | - |
| `type` | `"choice"` | `"score"` | `"noul"` |
| `instructions` | the question to decide | what to rate | the yes/no question or statement |
| `criteria` | map option → description (or `null`); up to 255 | ordered array of 2–10 level descriptions | optional `{true, false}` descriptions |

`instructions` and every entry in `criteria` accept a string, object, or array. Use an
object when a question needs supporting data or several fields:

```json
"instructions": {
  "question": "Is the resume for the same person as `potential_duplicate`?",
  "potential_duplicate": {"name": "Jon Smith", "location": "Oakland, CA"}
}
```

Use structure only when it separates guidance that would otherwise blur together. For
Choice options and Score levels, structured entries with the same field names (`what`,
`not_for`, `examples`) sharpen boundaries between similar options.

## Output: answers

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "intent": {
      "type": "choice",
      "choice": "refund",
      "probabilities": {"refund": 0.72, "exchange": 0.24, "other": 0.04},
      "confidence": 0.58
    },
    "severity": {
      "type": "score",
      "score": 1.43,
      "legend": {"0": "Cosmetic", "1": "Workaround exists", "2": "Blocking"},
      "probabilities": {"0": 0.0, "1": 0.57, "2": 0.43},
      "confidence": 0.35
    },
    "refund_requested": {
      "type": "noul",
      "noul": 0.93
    }
  },
  "usage": {"input_tokens": 328, "output_tokens": 34}
}
```

| Answer | Fields | How to read |
| - | - | - |
| Choice | `choice`, `probabilities`, `confidence` | `choice` = highest probability. `probabilities` sums to 1 across your options |
| Score | `score`, `legend`, `probabilities`, `confidence` | `score` = Σ level × probability, can land between levels. `legend` maps level numbers back to descriptions |
| Noul | `noul` | P(yes), 0–1. No separate confidence |

Every answer is constrained to the options or levels provided. Nothing has to be parsed
out of prose.

## Confidence

`confidence` summarizes how concentrated the distribution is. Act, review, or escalate on
it, per action. It is not correctness and not permission to act.

- Choice with `n` options and top probability `p_max`: `(p_max − 1/n) / (1 − 1/n)`
- Noul: `|2p − 1|` if a confidence-style number is needed
- Score: MAD-based over ordered levels, so probability on a neighbouring level costs
  less than on a level further away. Exact formula:
  https://docs.typesafe.ai/confidence.md

For ranking, sort by probabilities (a Noul's value or a Choice's top probability), not
confidence. Useful alternatives: top probability `p_max`, and top-to-second ratio
`p_max / p_second`.

## Python SDK

```sh
pip install typesafe-sdk        # Python >= 3.10
export TYPESAFE_API_KEY=...
```

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

client = TypeSafeClient()  # reads TYPESAFE_API_KEY; default model jev-latest

response = client.system_one(
    state={"ticket_message": "My flight was cancelled. Can I get a refund?"},
    questions={
        "intent": Choice(
            instructions="What is the main request in `ticket_message`?",
            criteria={
                "refund": "The customer wants money returned.",
                "rebooking": "The customer wants a replacement flight.",
                "information": "The customer asks for information only.",
            },
        ),
        "frustration": Score(
            instructions="How frustrated does the customer appear?",
            criteria=["Calm.", "Concerned but civil.", "Very angry."],
        ),
        "refund_requested": Noul(
            instructions="Does `ticket_message` request a refund?",
        ),
    },
)

print(response.answers["intent"].choice)
print(response.answers["intent"].confidence)
print(response.answers["frustration"].score)
print(response.answers["refund_requested"].noul)
```

- `AsyncTypeSafeClient` for async code, `await client.system_one(...)`.
- `response.nouls`, `response.choices`, `response.scores` are typed views; `answers`
  keys by question id.
- `response_model=` gives typed responses; `RetryPolicy` configures retries; clients can
  be pointed at AI gateways with `base_url` + `model`.
- One client per thread under heavy load; reuse clients; environment variables:
  `TYPESAFE_API_KEY`, `TYPESAFE_DEFAULT_MODEL`, `TYPESAFE_BASE_URL`, `TYPESAFE_LOG_LEVEL`.

## JavaScript SDK

```sh
npm install @typesafe-ai/sdk   # Node >= 20
```

```ts
import { choice, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const response = await client.systemOne({
  state: { document: "I was charged twice." },
  questions: {
    category: choice("What is this ticket about?", {
      billing: null,
      technical: null,
      other: null,
    }),
  },
});
console.log(response.answers.category.choice);
```

## Limits, errors, cost

| Item | Value |
| - | - |
| Model | `jev-latest` resolves to `jev-1.13.0`. Pin the version when thresholds were tuned on it |
| Context | 64k tokens per request; 32k for state plus the longest question |
| Latency | about 100 ms typical |
| Price | $0.042 per million input tokens; output tokens free |
| Rate limits | 100k tokens/s and 80 requests/s; `429` or `529` when exceeded |
| Errors | `401` bad key, `422` request validation, `429` rate limit, `529` overloaded |
| Retries | SDKs retry with backoff by default and honor `retry-after` |
| Token accounting | `usage.input_tokens` / `usage.output_tokens` per response |

Questions in one request are judged in parallel. Batching all questions about one state
into one call is the main cost and latency win; extra questions cost only their own
tokens.

## What makes questions work, and why

- **One narrow judgment per question.** Something a knowledgeable person could answer in
  a second given the right context. Broader questions return one number that cannot be
  inspected, tuned, or reweighted; decomposing keeps each judgment visible and lets code
  own how it is combined.
- **The instruction carries the meaning.** Question ids are for code and never reach the
  model, so a descriptive id is not a substitute for a complete question.
- **Phrase yes/no so a high value means the action.** Inverted true/false makes the
  answer easy to misread and the model handles it worse.
- **Describe situations, not degrees.** "Blocking issue, no workaround" gives the model
  something to match; "severity 2" does not. Level descriptions and option descriptions
  are the rubric, so they do most of the work.
- **Give the answer space an exit.** An `other` / `none` / `not stated` option lets the
  model report that the declared set does not fit instead of forcing a wrong pick, and a
  separate presence question answers "is it there at all" when that matters on its own.
- **Trust comes from evidence, not from the docs.** Test the question set on
  representative inputs; set thresholds from separation on your data and the cost of
  each error. Cookbook numbers are examples of the method, not values to adopt.
