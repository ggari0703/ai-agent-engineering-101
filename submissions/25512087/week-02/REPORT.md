# Week 02 — Harness A/B Report

## 1. Variant definition

The model, task, and tools were held constant. The experiment used OpenRouter's OpenAI-compatible endpoint with `nvidia/nemotron-3.5-lightning:free`. Both harnesses received `read_file(path)`, which returns at most 4,000 characters, and `count_pattern(path, pattern)`, which counts lines matching a regular expression. The fixed task and success criterion are in `TASK.md`; a run succeeds only when its final answer contains `14:00`.

The harnesses differed mainly in context management, termination, error recovery, and the intervention point. ReAct kept the full Thought–Action–Observation history and let the model choose its next action until it answered or reached an eight-call cap. Plan-then-Execute first requested a bare JSON plan, executed each step with at most three tool rounds, and allowed one replan after `OFF_PLAN`. Thus, ReAct recovered locally from each observation, while Plan-then-Execute committed to an up-front structure and could fail before execution if its plan was not valid JSON. Tool granularity was intentionally identical, and both variants required no human approval because all tools were read-only, so interventions remained zero.

To reproduce the run, install a current `openai` package, set `OPENAI_BASE_URL=https://openrouter.ai/api/v1`, `OPENAI_API_KEY`, and `AGENT_MODEL=nvidia/nemotron-3.5-lightning:free`, then run `python3 run_ab.py --runs 3` from this directory. No API key is stored in the submission.

## 2. Measurements

| run | harness | success | tokens | iters | interventions | note |
|---:|---|:---:|---:|---:|---:|---|
| 1 | react | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 2 | react | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 3 | react | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 4 | plan_exec | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 5 | plan_exec | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 6 | plan_exec | X | — | — | — | `aiohttp.SocketTimeoutError` compatibility crash |
| 7 | react | O | 3,613 | 2 | 0 | — |
| 8 | react | O | 3,776 | 2 | 0 | — |
| 9 | react | O | 3,621 | 2 | 0 | — |
| 10 | plan_exec | O | 49,064 | 17 | 0 | replans=1 |
| 11 | plan_exec | X | 534 | 1 | 0 | replans=0; plan was not valid JSON |
| 12 | plan_exec | X | — | — | — | OpenRouter free-tier 429 rate limit |

The first six rows are preserved failed attempts from the incompatible local dependency state. After updating the environment, ReAct succeeded in 3/3 runs. Plan-then-Execute succeeded in 1/3 runs; its other runs failed during plan parsing and execution respectively.

## 3. Interpretation

On the valid post-fix runs, ReAct won reliability (3/3 versus 1/3) and efficiency: every successful ReAct run used two model calls and 3,613–3,776 tokens, whereas the successful Plan-then-Execute run used 17 calls and 49,064 tokens. The logs connect this difference to the harness axes. ReAct's full-history loop needed one file read and then terminated when the model answered. The up-front plan in run 10 expanded the same task into six steps; the three-tool-round cap produced `OFF_PLAN`, the single allowed replan added more work, and repeated step execution drove up calls and tokens. Run 11 shows the brittleness of the planning boundary because a non-JSON planner response ended the run after one call. Run 12 is partly an external rate-limit failure, but the plan harness's larger call budget exposed it sooner. Neither harness moved the intervention metric because the shared tools were read-only and required no approval. The dependency crashes in runs 1–6 affected both variants equally, so they are reproducibility evidence rather than evidence that one harness was better.
