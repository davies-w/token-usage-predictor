# Where to get prompts to experiment with

## Best first: your own traffic

If you have application logs or request records, that is the most representative sample and the only place you can get paired actual usage for calibration. Everything public is a fallback for when you don't have enough of your own data yet. Start here.

## Public sources by category

Realistic chat / instruction traffic (closest to production):
- LMSYS-Chat-1M
- WildChat-1M
- ShareGPT
- OpenAssistant OASST (oasst1 / oasst2)
- Alpaca
- Databricks Dolly-15k

QA / retrieval (what S3 and SSJF-style work used):
- Natural Questions
- TriviaQA
- SQuAD
- HotpotQA
- MS MARCO
- The Pile

Reasoning-heavy (to stress the hard case):
- GSM8K
- MATH
- MMLU and MMLU-Pro
- GPQA
- Big-Bench Hard
- AIME
- ZebraLogic
- MuSR

Code:
- HumanEval
- MBPP
- APPS
- CodeContests
- SWE-bench

Long-form:
- LongBench
- ELI5
- NarrativeQA

Most of these are on the Hugging Face datasets hub. Several papers also release their own artifacts and prompt sets; S3 used NQ and the Pile, Response Length Perception trained on Vicuna-style prompts (likely ShareGPT-derived).

## Practical cautions

- **Stratify by task type and difficulty.** You specifically need the long tail of expensive requests, not a pile of easy short ones.
- **Bare prompts aren't realistic.** Public sets usually lack system prompts and conversation history. Wrap them in your real system instructions and simulated multi-turn history, because those change input cost and often output length.
- **Include tools/agents if you use them.** Single-shot QA will badly understate usage for tool-using or agentic workloads.
- **Size.** A few thousand prompts per target model is enough to train a bucket classifier; paired actual usage is what you must collect. Tens of thousands help if training from scratch.
- **Calibration needs ground truth.** You must run the prompts through the target model and record actual usage (input, output, and reasoning where applicable).
- **Licensing and PII.** LMSYS-Chat-1M and WildChat contain real user content. Check terms before use and before committing anything derived from them.

## Suggested starter mix

If you don't have your own traffic yet, something like:
- 40% realistic chat/instruction (LMSYS or WildChat or ShareGPT)
- 30% reasoning-heavy (GSM8K, MATH, MMLU-Pro)
- 20% QA/retrieval (Natural Questions, TriviaQA)
- 10% code (HumanEval, MBPP)

Then wrap each in a representative system prompt and, for some fraction, a multi-turn history. Log actual usage from the target model to serve as labels and to derive bucket boundaries.

Keep the raw prompts and any personal data out of the repo; store only schemas, loader code, and derived statistics.
