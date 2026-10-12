# Finding fit: patterns, examples, evaluation

Starting points for the thinking in `SKILL.md`, not a procedure to follow
literally. Tools, languages, and layouts differ per repo; adapt.

## Search patterns

### LLM call sites

```sh
rg -n "chat\.completions\.create|messages\.create|responses\.create|generate_content|generateContent|litellm|langchain|cohere|ollama"
rg -n "from (openai|anthropic|google\.generativeai|mistralai|groq|cohere) import|import (openai|anthropic)"
rg -n "\"(gpt-|claude-|gemini-|llama|mistral|command-r|o[13]-)|'(gpt-|claude-|gemini-|llama|mistral|command-r|o[13]-)"
```

### Prompt artifacts

```sh
rg -n "You are |You're a|SYSTEM_PROMPT|system_prompt|PROMPT_TEMPLATE|prompt_template"
rg --files -g '*.prompt' -g 'prompts/**' -g '**/*.jinja' -g '**/*.handlebars' -g '**/ai/**' -g '**/llm/**'
```

### Parsing model output (the strongest smell)

```sh
rg -n "json\.loads\((response|resp|result|text|content|message|completion|output)"
rg -n "extract_json|parse_json|clean_json|re\.search\(.*(completion|response)"
rg -n "while .*(json|parse|valid)|retry.*(json|parse|format)" --glob '*.py'
```

A retry-until-valid-JSON loop is a decision model fighting its own output
format; typing the answers removes the fight.

### Hand-rolled semantic heuristics

```sh
rg -n "sentiment|urgency|priority|intent|spam|toxic|escalat|refund|complaint|jailbreak|injection" --glob '*.{py,js,ts,go,java,kt,rb}'
rg -n "in (text|body|message|subject|prompt)\.lower\(\)|\.includes\(|\.test\(" --glob '*.{py,js,ts}'
```

Read matches with an if-chain or keyword list over free text. Deterministic
lookups (status codes, IDs, enum membership, arithmetic) belong to code. A
keyword list standing in for "does this message mean X" is a candidate.

### LLM-as-judge

```sh
rg -n "rate (this|the)|score (this|the)|evaluate whether|classify (this|the)|return JSON|respond with JSON"
```

## Reading a match: worked reasoning

Each example shows the observation and the reasoning, not a rule.

- `if json.loads(resp)["intent"] == "refund"` — the consumer needs one of a
  known set. The parse exists only because the interface is text. A Choice over
  the intent options, plus `other`, removes the parse and returns confidence
  for routing.
- A retry loop until the model returns valid JSON — the consumer needs fields
  the code can trust. Typed questions per field make validity structural.
- `score = 0.6 * keyword_hits(urgency) + 0.4 * exclamation_count` — someone is
  approximating a judgment with proxies. A Noul for urgency and a Score for
  frustration, combined by the same weights in code, is more direct; the
  weights remain reviewable policy.
- `if "refund" in text.lower()` — literal matching misses paraphrases ("I want
  my money back"). A Noul handles the meaning; keep any deterministic checks
  that remain useful.
- An LLM judge asked "is this citation supported?" — partly checkable, partly
  holistic. A string search for the quote plus a Choice relation question
  covers the checkable part at a fraction of the cost; the holistic residue can
  stay with the LLM.
- Embedding search returns candidates in rough order — the sort is a judgment
  in disguise. One Noul per shortlisted pair, sorted by the value, re-ranks
  where it matters.
- "Is this whole record good?" — broad, and failures are not localized. One
  Noul per field, framed so yes means a problem, then escalate if any fires. A
  max gate keeps one confident red flag from being averaged away.
- A draft-summarisation prompt that also labels the summary — two consumers, one
  output. Follow each output separately: the prose stays with the LLM, the
  label becomes a Choice.

## Decomposition patterns

Useful shapes, each with the idea behind it.

- **Compound judgment** → one question per factor, then a weighted sum in code.
  Weights live in code so priorities change without rewriting a prompt.
- **Broad classification** → a single Choice with every option the code can
  handle, plus `other`. For deep taxonomies, one Choice per level and walk the
  tree in code, keeping several candidate paths alive when probabilities are
  close.
- **Unknown answer space** → code gathers candidates (regex, a known roster, or
  a model that proposes them), then a Choice over those spans with a `none`
  escape hatch. The model selects; code copies the picked span verbatim.
- **Record verification** → per-field Nouls, one reason per question, framed so
  a high value means a problem. Escalate on any fire; keep the per-field
  answers visible to see which field failed.
- **Ranking** → one Noul per query-candidate pair, sort by the value. One
  request per pair; run pairs concurrently.
- **No-match handling** → a `none` / `not stated` / `other` option whenever the
  set might not cover the input, and a separate presence question when "is it
  there at all" is independently useful.
- **Speculative fan-out** → ask for branch-specific answers before the branch
  is known. Extra questions in the same request cost only their own tokens.

## Model limitations as design inputs

Observed behavior of `jev-1.13`; newer versions may differ, and the live page is
https://docs.typesafe.ai/model-jaggedness/jev-1.13.md. These shape the design;
they rarely rule a use case out.

| Limitation | What it looks like | Usual design response |
| - | - | - |
| Literal reading | Answers the words written, not the intent behind them | State the exact condition; move boundary cases into criteria |
| Math and counting | Unreliable counts, sums, numeric interpolation | Compute in code. To count matches, iterate in code and ask one question per item |
| Dates and times | Orders and compares dates poorly | Extract parts as Choices (month/day/year), resolve and compare in code |
| Indirection | Double negatives and multi-hop questions lose accuracy | Shorten the question, or split it into two literal ones |
| Large irrelevant state | Distractors reduce accuracy | Filter in code first; send only the fields the questions need |
| Adversarial content | Injected instructions or self-serving text can move answers | Write precise criteria, test injections, keep a review path |
| Instruction/criteria conflict | Contradictory phrasing confuses the answer | Make criteria an extension of the instruction; avoid inverted true/false |
| Option order | Choice leans toward the first option in some cases | Reorder once to check; keep the validated order stable |
| Generation | Not trained to produce text | Use a generative model for prose; Jev can select the shape of it |

## Evaluation ideas

Calibration is empirical, so a small experiment before production beats
reasoning about it.

- Collect 20–100 representative inputs, including cases the current mechanism
  gets wrong.
- One request per input; print each answer with probabilities and confidence.
- Look for separation: do correct and incorrect cases receive different
  values? A signal that is high on errors and low on correct cases is usable; a
  flat one is not.
- Pick thresholds from the cost of each error. Starting conservative, with the
  middle routed to review or the old mechanism, is a valid first version.
- Re-run a sample for stability, reorder Choice options once for order bias,
  and re-read the questions for literal-reading gaps.
- Record the validated questions and thresholds together, with a note about
  the data they were validated on.

## Example report format

A shape that has worked; adapt to the team's review habits. The reasoning per
site matters more than the columns.

```markdown
## Jev scan — <repo> — <date>

| # | Location | Current mechanism | Consumer | Disposition | Proposed questions | Gate | Priority |
| - | -------- | ----------------- | -------- | ----------- | ------------------ | ---- | -------- |
| 1 | src/ai/triage.py:42 | Claude call -> json.loads | `if intent ==` | Jev | `intent` (Choice) | confidence >= 0.6 else review | high |
| 2 | src/ai/reply.py:88 | Claude call | shown to user | keep LLM | - | - | - |
| 3 | src/util/priority.py:12 | keyword list | `if urgent:` | Jev | `is_urgent` (Noul) | >= 0.7 escalate | medium |

### Candidate 1: ticket intent

- State: `{ticket.message, ticket.sender, order}` — the full order history is
  left out because no question needs it
- Questions:
  - `intent` (choice): "What is the customer's primary request in
    `ticket.message`?" criteria: refund, exchange, information
  - `is_urgent` (noul): "Does `ticket.message` convey time pressure?"
- Consumption: `if intent.confidence < 0.6: route_to_human()`, else switch on
  `intent.choice`; `is_urgent.noul >= 0.7` sets priority
- Fallback: low confidence and `other` go to human review
- Evidence: inputs that currently mis-parse, with the expected intent and
  urgency for each
```
