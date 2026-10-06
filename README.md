# Predicting token usage before an LLM request

A brief review of five papers and a practical approach to coarse estimates

A useful forecast does not need to predict an exact token count. The research supports training a small model to estimate output length from a prompt, especially as a range or usage category. Expected spending can then be calculated using the target model's prices. The main unresolved question is how well these estimates transfer to long, hidden reasoning and unfamiliar workloads.

# What the five papers show

## 1 PALACE 2025

Predictive Auditing of Hidden Tokens in LLM APIs via Reasoning Length Estimation estimates hidden reasoning tokens from a prompt and the target model's completed answer. It uses domain-specific adaptation and routing. With Qwen2.5-3B, 87.28% of general predictions and about 59-62% of math, coding and medical predictions fall within 33% of actual reasoning usage. Its role is retrospective auditing; it does not establish prompt-only forecast accuracy. The useful idea is learning reasoning usage from semantic information and target-specific examples.

[PALACE paper](https://arxiv.org/html/2508.00912v1)

## 2 OUTLETS 2026

Output-Length Prediction from Speculative Decoding Backbones reuses a small draft decoder and target-model internal states to predict generation length. It estimates length before generation and updates the forecast during generation. On Qwen3-30B-A3B, static mean absolute error is approximately 167-187 tokens for average lengths of 726-1,054. Outputs longer than 2,048 tokens are excluded. This supports length prediction when internal access is available; it is not an independent predictor for an opaque API.

[OUTLETS paper](https://arxiv.org/html/2609.01068v1)

## 3 SSJF 2024

Efficient Interactive LLM Serving with Proxy Model-based Sequence Length Prediction trains BERT-base to predict another model's output length from the input query. Predictions inform request scheduling. It reports 61.5% accuracy when assessed across five length categories, using data filtered to responses of 1-512 tokens. This is the closest precedent for an inexpensive, independent prompt-only estimator. Its accuracy is category accuracy, not a percentage error in total cost.

[SSJF paper](https://arxiv.org/html/2404.08509)

## 4 S3 2023

S3 Increasing GPU Utilization during Generative Inference for Higher Throughput uses a 66M-parameter DistilBERT model to assign prompts to ten output-length buckets. It reports 77.13% bucket accuracy on Natural Questions and 65.6% on Pile, with 3.7 ms predictor latency on an A100. The estimates guide memory allocation and batching. This provides a concrete example of cheap, coarse prediction, although these results do not establish accuracy for hidden reasoning or frontier API bills.

[S3 paper](https://arxiv.org/html/2306.06000)

## 5 Response Length Perception 2023

Response Length Perception and Sequence Scheduling An LLM-Empowered LLM Inference Pipeline trains a model to produce a numerical length estimate without generating the full answer. Instruction tuning reduces reported prediction error from 193 to 63 tokens. Training uses 10,000 prompts with four generations per prompt, taking the maximum length as the label. Alternative models, including GPT-2, perform worse at predicting Vicuna's length than Vicuna's self-prediction. It supports explicit length prediction while showing that cross-model transfer needs validation.

[Response Length Perception paper](https://arxiv.org/abs/2305.13144)

# A practical estimator

The following is a proposed implementation, rather than a performance claim from the papers. Begin with one target model and a representative workload. Include the full request context and settings: system instructions, conversation history, reasoning effort, output format and output limit. Predict billed generated tokens, including reasoning, rather than visible answer length alone.

## Start with ranges and a historical baseline

Collect actual target usage for representative requests. First estimate usage from the average for each task type and configuration. Then train a small text encoder or classifier to predict usage buckets from the prompt. Illustrative buckets might be under 500, 500-2,000, 2,000-8,000 and over 8,000 generated tokens; choose boundaries from observed usage and product needs, not from these example numbers.

For an expected token count, combine each bucket's probability with its observed mean usage. For example, a 70% chance of a bucket averaging 1,000 tokens and a 30% chance of one averaging 5,000 yields 2,200 expected tokens. Estimate the open-ended bucket from real data; it has no finite midpoint. Report an upper budget estimate alongside the expectation, calibrated against held-out actual usage.

# Using a cheap model as a probe

A second version can run the prompt through a cheap model and add its response, token usage and completion status as features. Train the mapping using paired cheap-model and target-model runs. A fixed multiplier is a hypothesis to test, not an established scaling law: the smaller model may struggle for longer, give up earlier, or take a different route from the target.

Compare this probe against the prompt-only predictor on unseen requests. Keep it only if the improvement justifies the extra charge and latency. None of these five papers establishes a universal conversion from a cheap model's complete answer length to a frontier model's total billed usage.

# Converting tokens into expected cost

For a text request with one generated-token rate, expected cost equals known input cost plus expected billed generated tokens multiplied by the output price per token. Include the probe's cost when it is used. Apply cached-input rates or other charges separately where relevant. Training on tokens instead of dollars makes it easier to update the estimator when prices change.

For workflows involving tools, retries or multiple model calls, total cost requires an additional forecast of call counts and future context sizes. A single-response length predictor covers only one part of that workflow.

# How to tell whether it is useful

Hold out whole prompt families and later requests to avoid testing on near-duplicates. Compare with the historical task-average baseline. Track absolute token error, total spending bias, bucket accuracy and upper-budget coverage. Also inspect the most expensive requests: acceptable average performance can conceal costly underestimates.

Repeat a subset of target requests to measure usage variability. Calibrate expected usage and upper ranges separately; a median or maximum-length prediction is not automatically an expected value. Recheck calibration after target-model or reasoning-setting changes. Aggregate errors may cancel on one workload and become systematic on another.

# Recommended starting point

Build an SSJF or S3-style prompt-only predictor for one target model, with a few meaningful usage ranges and a calibrated expected count. This is a defensible path to providing some information before a request runs. Present the result as an estimate with uncertainty, and add the cheap-model probe only if paired testing demonstrates a useful gain. PALACE supplies ideas for reasoning-specific adaptation; OUTLETS supplies evidence for richer prediction when model internals are accessible.

The published metrics above use different tasks, length limits and definitions. They are evidence of feasibility and should not be read as a common accuracy benchmark or a guarantee for a new workload.
