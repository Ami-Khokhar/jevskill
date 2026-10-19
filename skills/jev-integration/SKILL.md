---
name: jev-integration
license: MIT
description: >-
  Think through where TypeSafe Jev belongs in a product or codebase, and
  integrate it well. Use when asked where Jev or TypeSafe fits; when weighing a
  model call against typed decisions; when replacing fragile LLM
  prompt-and-parse calls, LLM-as-judge checks, or hand-rolled text heuristics;
  when a feature needs classification, routing, ranking, verification,
  extraction into a closed set, or confidence-gated escalation; or when
  model-call latency and cost matter. Covers the philosophy, the thinking
  method for finding fit, and how to call Jev: inputs (state plus questions)
  and outputs (typed answers, probabilities, confidence).
---

# Jev integration

Jev is a System One decision model from TypeSafe: send a `state` and typed
`questions`, get typed `answers` with calibrated probabilities, usually in about
100 ms. It does not write text, hold a conversation, or reason in steps. It makes
narrow judgments that software consumes directly.

This skill is about thinking, not a fixed procedure. It explains the idea behind
Jev, a way to find where it belongs, and principles for integrating it. The
mechanics — language, module layout, how findings are reported, whether to batch
— adapt to the project. The API facts live in `references/jev-contract.md`, with
search patterns, examples, and evaluation ideas in
`references/finding-fit.md`. Current API details are always in the live docs:
https://docs.typesafe.ai/llms.txt. If the `typesafe-ai` skill is installed, use
it for API specifics and cookbooks.

## The idea underneath

Software constantly needs judgments about language: is this urgent, which team
should see it, does this evidence support the claim, how relevant is this
passage, can this extraction be trusted. Today those judgments usually come from
one of two places. An LLM call followed by parsing: text generated, coaxed into
a shape, re-validated. Or a hand-rolled heuristic over text: keyword lists and
if-chains that encode one author's guesses and decay.

Jev takes a third position: **the decision is the interface.** The possible
answers are declared up front — options, levels, yes/no — so code receives a
value it can branch on, sort by, or threshold without a parser, plus a
probability distribution that says how contested the judgment was. Three
properties follow, and they are the reason to reach for it:

- **Typed by construction.** Every answer is constrained to what you declared.
  Nothing is recovered from prose.
- **Uncertainty is an output, not an error.** Confidence and probabilities let
  the system route the unsure instead of pretending.
- **Cheap and fast enough to be infrastructure.** Around 100 ms and per-token
  pricing means judgments can live inside request paths, interfaces, and
  per-row pipelines, not only background jobs.

That yields a division of labor: **code owns the workflow** — control flow,
policy, arithmetic, side effects — and **the model supplies common sense** where
code would otherwise guess. The companion idea: one broad call becomes several
narrow questions asked together against the same state, combined by code.

Because the cost is low, "where does Jev fit" is rarely just "which LLM call do
we swap". It is also "which decisions are currently missing, guessed, or too
expensive to make volume". Start from what the product should show, select,
change, or hand off, and work backwards to the judgments it needs.

## Finding where it fits

Two lenses. Useful alone, stronger together.

### Lens 1: start from the product behavior

Describe what the application will show, select, change, or hand off. For each
of those, ask which parts require understanding language at all, and which of
those are currently approximated, dropped, or escalated to a human because a
judgment was unavailable or too expensive. A product that routes everything to
a person, or silently drops edge cases, is describing missing judgments. Work
backwards from the behavior to the questions.

### Lens 2: start from the code

Existing code shows where judgment is outsourced or faked. Look for:

- Model calls whose results are parsed, retried until valid, or branched on.
- LLM-as-judge checks that gate an action.
- Keyword lists, regexes, and if-chains making semantic calls over free text.
- Checks that run rarely because each one is expensive, or do not run at
  volume at all.
- Human fallbacks for cases the system cannot decide.

`references/finding-fit.md` has search patterns and worked examples.

### Decide from the consumer backwards

For each candidate, ask what the consumer of the output actually needs. That
determines the primitive, because the primitives mean different things:

- A selection among meanings → **Choice**. The distribution compares the
  options; confidence is how concentrated the pick is.
- Whether a condition holds → **Noul**. The value is the probability of yes,
  and it doubles as the degree of belief.
- A position along an ordered dimension → **Score**. The value is
  probability-weighted across levels you describe in words.
- Where something is in an image → **Box** (beta, Jev 1.14). Four edges as
  fractions of the image, plus confidence; pair it with a Noul for presence.

If the consumer wants prose, keep the LLM. If it wants a number code can
compute, write code. Read `references/jev-contract.md` for the exact shapes.

Known model limitations are design inputs, not reasons to give up: Jev reads
literally, and does not count, do arithmetic, compare dates, follow multi-hop
indirection, or generate. Accuracy drops with large irrelevant states, and
adversarial state can move answers. The usual response is decomposition — code
does the deterministic half (counts, dates, math, lookup), the model does the
semantic half. The full list is in `references/finding-fit.md`.

### Let economics choose placement and shape

Rank opportunities by how often the decision occurs, what a wrong answer costs,
and how fragile the current mechanism is. Placement matters as much as the
model swap: a judgment in the request path needs the ~100 ms guarantee, a
per-document scoring job at volume needs the token price. Questions about one
state run in parallel within a single request, so batching every question a
state might need — including ones that only matter on some paths — is where
most of the speed and cost advantage comes from. A decision that was too
expensive to make can become routine.

## Shaping the judgments

These principles apply whether you are designing questions or reviewing an
agent's draft.

- **Decompose until each question is a gut check** — something a knowledgeable
  person could answer in a second given the right context. A question that
  hides several judgments returns one number that cannot be inspected or
  tuned; atomic answers compose and can be reweighted.
- **The instruction carries the meaning; criteria define the answer space.**
  Question ids are for your code and never reach the model, so write the
  complete question in `instructions`. Structure in `instructions` and
  `criteria` earns its place when it separates guidance that would otherwise
  blur — supporting data, boundary cases, contrasts, examples.
- **Answer spaces should be complete.** Add a no-match option when the declared
  set might not cover the input, and check candidate coverage when selecting a
  value: the model cannot choose something you did not offer.
- **Ask everything together, then let code filter.** Questions about the same
  state are independent and judged in parallel. A second request is justified
  only when code needs an answer to build the next state, fetch evidence, or
  choose the next options.
- **Compose in code with explicit policy.** Normalize scores before weighting
  them; keep thresholds per action and per risk; prefer a max-style gate over a
  mean when one serious violation should escalate. Policy in code can be read,
  reviewed, tested, and changed; policy buried in a prompt cannot.

Worked decompositions and structured question examples are in
`references/finding-fit.md`.

## Integrating

The principles, each with the reason it matters:

1. **The question set and its thresholds are the policy of the AI step.** Keep
   them together, named, and reviewable, like configuration. Question quality
   is the main lever, and agents write weak questions — plan to iterate with
   the owner.
2. **Code owns control flow.** When the model starts choosing its own next
   step, you have built an agent, with agent failure modes. Keep Jev on the
   judgment, keep the loop in code.
3. **Learn the model's behavior on your data before trusting thresholds.** A
   short probe over representative cases shows separation and calibration.
   Interface guarantees are about types, never about truth. Cookbook
   thresholds are examples, not values to adopt.
4. **Route the unsure.** Use confidence to act, review, or escalate, with
   thresholds that differ by the cost of being wrong. A middle value going to
   a person or to a second check is the system working, not failing.
5. **Respect the boundary facts.** State is data, not instructions; keep
   secrets out of it; the API key stays server-side. These are properties of
   the model and the product, not style choices.
6. **Verify in the real runtime and measure.** Split failures by cause —
   missing evidence in state, a bad question, a bad threshold, or a code bug —
   because the fixes differ.

Mechanics worth adapting rather than copying: sync or async client, SDK or
raw HTTP, where the question module lives, how many speculative questions are
worth their tokens, and whether the integration ships behind a review path
first.

## Reporting what you find

The value of a scan is the reasoning, not a template. Capture, per site: where
it is, the current mechanism, what consumes the output, your disposition and
why, the questions you would ask, the gate or fallback, and the evidence that
would prove the change. This record lets a reviewer check the reasoning even
when they disagree with the conclusion. An example table is in
`references/finding-fit.md`; shape yours to the team's habits.

## References

- `references/jev-contract.md` — inputs, outputs, primitives, confidence, SDKs,
  limits, and what makes questions work, with reasons.
- `references/finding-fit.md` — search patterns, reasoning examples,
  decomposition patterns, model limitations, evaluation ideas, and an example
  report.
