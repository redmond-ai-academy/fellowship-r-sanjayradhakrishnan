# AI_LOG: how I used my coding assistant

Be specific and honest. This is not a confession. It is how you learn to work with AI and still own the result.

| # | What I asked it to do | What it did | What I kept | What I changed or rejected, and why |
|---|---|---|---|---|
| 1 | Explain the triage architecture, codebase files, and build a master roadmap. | Explained medical vs support triage, Pydantic data schemas, `FakeClient` testing, and broke the project into 7 milestones. | The master step-by-step plan, starting with git branching and baseline test failures. | Paused the assistant and asked for detailed line-by-line function call diagrams before writing any code to ensure I understood every concept. |
| 2 | Help build Step 1 (Happy path validation) and Step 2 (Audit tracing). | Implemented `Triage.model_validate_json()` and the `write_trace()` helper with `time.perf_counter()`. | The append-only `.jsonl` logging and millisecond latency tracking. | Stopped to ask why `.jsonl` was used instead of standard `.json` and where `tmp_path` files are stored on disk. |
| 3 | Implement Step 3 retry logic and debug API errors during real ticket runs. | Implemented the 2-attempt loop in `triage.py` catching `ValidationError`, and added exponential backoff in `llm.py`. | The informed retry prompt passing Pydantic error details back to Gemini, and the backoff in `llm.py`. | Evaluated two options for rate limits and rejected a fixed 12-second delay across all tickets in favor of smart retry with backoff in `llm.py`. |

## Something the AI got wrong
When we first ran all 12 tickets through `run.py --eval`, the run failed after 4 tickets because the assistant didn't initially account for Google AI Studio's free-tier rate limits (5 requests per minute and 20 requests per day). I noticed when the terminal returned 429 and 503 error codes, which led us to investigate and build exponential backoff in `llm.py`.

## One thing I would do differently next week
Check third-party API quotas and rate limits before running batch scripts, and continue having the assistant explain the data flow between files before writing code.
