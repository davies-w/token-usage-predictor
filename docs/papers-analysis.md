# Analysis of the five papers

Sources: the project README (a review brief) and each paper's abstract on arXiv.
Links and IDs:

| # | Name | Year | arXiv |
|---|------|------|-------|
| 1 | PALACE | 2025 | 2508.00912 |
| 2 | OUTLETS | 2026 | 2609.01068 |
| 3 | SSJF | 2024 | 2404.08509 |
| 4 | S3 | 2023 | 2306.06000 |
| 5 | Response Length Perception | 2023 | 2305.13144 |

## Grouping by what they assume you can access

**Prompt-only predictors** (usable before an opaque API call):
- SSJF: BERT-base proxy predicts the target's output length from the query alone. 61.5% across five length categories, on data filtered to responses of 1-512 tokens. Built for request scheduling.
- S3: 66M DistilBERT assigns prompts to ten output-length buckets. 77.13% on Natural Questions, 65.6% on Pile, 3.7 ms predictor latency on an A100. Built for memory/batching.
- Response Length Perception: trains a model to emit a numeric length with no generation. Instruction tuning cuts reported error from 193 to 63 tokens. Labels are the max of four generations per prompt. Cross-model transfer is weak.

**Internal-access predictor**:
- OUTLETS: reuses the speculative-decoding draft decoder plus target internal states. MAE ~167-187 tokens at mean lengths 726-1054, excluding outputs over 2048 tokens. Requires model internals, so it does not apply to a black-box API.

**Post-hoc reasoning auditor**:
- PALACE: estimates hidden reasoning tokens from the prompt and the target's completed answer, with domain-specific adaptation and routing. Within 33% of actual reasoning usage for 87.28% of general predictions, but only ~59-62% for math, coding, and medical. Retrospective: needs the finished answer, so it is not a prompt-only forecast.

## Cross-cutting reasoning

1. **Convergence on one recipe.** The three prompt-only papers independently land on the same approach: a small text encoder, coarse usage buckets, cheap and fast (3.7 ms in S3). That is the only one of the three approaches that works when you have no model internals and don't want to pay for a target call.

2. **All evidence is on short outputs.** SSJF caps at 512 tokens; S3 and Response Length Perception operate on QA/instruction data. None tests long chain-of-thought. So we have almost no direct evidence for the case that matters most: long, reasoning-heavy generations.

3. **Hidden reasoning is the gap.** For reasoning models, billed output includes reasoning the user never sees. The prompt-only papers don't model it. PALACE does, but only after the fact and with materially lower accuracy outside general domains. Pre-call billing for reasoning-heavy models is the genuinely unsolved part; expect wide intervals there.

4. **Length is model-specific.** Response Length Perception shows self-prediction beats cross-model prediction, and the brief notes a fixed multiplier between models is a hypothesis, not a scaling law. So: one predictor per target model and reasoning setting, retrained or recalibrated when those change.

5. **The metrics don't match our goal.** 61.5% is category accuracy, 77% is bucket accuracy, MAE is in tokens. None reports calibrated intervals or cost error. Bucket accuracy is a useful component but not the deliverable.

6. **The brief already recommends the defensible path:** an SSJF/S3-style prompt-only predictor for one target model, with a few meaningful usage ranges and a calibrated expected count, presented with uncertainty. Add a cheap-model probe only if paired testing proves a gain. PALACE informs reasoning-specific adaptation; OUTLETS informs what becomes possible with internals.

## Implications for a pre-request cost estimator

- **Baseline first.** Per-task-type historical mean. Any predictor must beat it to be worth its complexity.
- **Model.** A DistilBERT/BERT bucket classifier over quantile-derived buckets. The paper buckets (<500, 500-2,000, 2,000-8,000, >8,000) are illustrative; choose boundaries from observed usage and product needs. The top bucket is open-ended and has no midpoint; estimate it from real data.
- **Combine probabilities with means.** Expected tokens = sum over buckets of (bucket probability times observed bucket mean). Example from the brief: 70% chance of a 1,000-token bucket and 30% of a 5,000-token bucket yields 2,200 expected tokens.
- **Ship an upper bound.** Report a budget estimate alongside the expectation, calibrated to cover the expensive tail at a stated rate.
- **Cheap-model probe (optional).** Run the prompt through a small model; add its response, token usage, and completion status as features. Train on paired cheap-model and target-model runs. Keep it only if it beats prompt-only on unseen requests enough to justify the extra charge and latency. No universal conversion from a cheap model's length to a frontier model's billed usage is established.
- **Train on tokens, not dollars.** Convert to cost at query time using the target model's prices, so price changes don't invalidate the model. Include the probe's cost when used; apply cached-input rates separately.
- **Workflows complicate this.** Tools, retries, and multiple calls need a separate forecast of call counts and future context sizes. A single-response predictor covers only one part.
- **Evaluation.** Hold out whole prompt families and later requests to avoid testing on near-duplicates. Compare against the historical task-average baseline. Track absolute token error, spending bias, bucket accuracy, and upper-budget coverage. Inspect the most expensive requests, because good averages can hide costly underestimates. Repeat a subset of identical requests to measure usage variability; calibrate expected usage and upper ranges separately; recheck after model or reasoning-setting changes.

## Open questions to resolve before building

- **Which target model(s), and do they reason?** If we start with a non-reasoning model, hidden reasoning drops out and the problem is much easier. If reasoning, interval width is the central design question.
- **What is the actual decision?** A pre-call budget check, a routing decision, a user-facing estimate, or spend forecasting? The required accuracy and the cost of error differ a lot.
- **How much history do we have?** Bucket boundaries and the open-ended bucket need observed usage. With little data, we start with the per-task-type mean and grow from there.
- **Single-turn or multi-turn?** Conversation history and system prompts materially change input cost and often output length.
- **Tool use?** If yes, the response-length predictor is a small part of total cost.
- **Cost of being wrong, in each direction.** Underestimating a budget is usually worse than overestimating; that sets the calibration target.
