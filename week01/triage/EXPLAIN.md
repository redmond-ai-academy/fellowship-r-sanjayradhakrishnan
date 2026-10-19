# EXPLAIN: Ticket Triage

Answer first. Write for a mentor who has not seen your code. Short sentences. Replace every line in angle brackets.

## 1. What it does, in one sentence
`triage()` reads a customer support ticket, prompts Gemini for structured JSON, validates it against a strict Pydantic schema with an informed retry on failure, and records an audit trace for every attempt.

## 2. Walk-through of every line I did not type myself
For each block of code that you did not write (from your assistant, a tutorial or the starter), say what it does and why it is there. If you cannot, that block is not ready to submit.

| Where (file and lines) | What it does | Why it is there |
|---|---|---|
| `triage.py`, lines 22–48 | `write_trace()` creates the trace directory if missing, builds a 7-key dictionary (timestamp, model, attempt, latency_ms, ok, error, input_chars), and appends it as a single JSON line to `YYYY-MM-DD.jsonl`. | Provides production observability, latency tracking, and an append-only audit trail without risking file corruption or losing logs. |
| `triage.py`, lines 133–178 | A `for attempt in (1, 2):` loop that times the LLM call, validates output using `Triage.model_validate_json()`, catches `ValidationError`/`ValueError`, feeds back the exact error message on attempt 2, and raises `TriageError` if the second attempt fails. | Enforces self-correction on hallucinated/invalid JSON, strictly caps retries at two, and guarantees unvalidated data is never returned to the system. |
| `llm.py`, lines 67–85 | Adds an exponential backoff loop inside `GeminiClient.generate_json` that catches HTTP `429` (rate limits), `503` (high demand spikes), and connection errors, sleeping before retrying. | Protects the backend against transient API errors and rate-limiting on free-tier quotas so batch evaluation can complete reliably. |

## 3. Why do outputs vary?
LLMs are probabilistic token predictors that calculate a probability distribution for each next token rather than using deterministic lookup tables. 

Temperature scales the probability distribution before sampling: at temperature 1.0, the distribution is flattened, allowing the model to sample lower-probability, diverse word choices (producing 5 completely different summaries across our 5 runs). At temperature 0, the model favors the highest-probability paths, producing consistent labels and focused phrasing across runs.

## 4. What the retry protects against, and what it does not
- Protects against: Malformed JSON syntax, hallucinated enum categories (like `"urgent"` instead of `"bug"`), invalid priority strings, missing required fields, and summary text exceeding the 25-word limit by explicitly showing the model its previous output and Pydantic's validation error.
- Does not protect against: Semantically plausible misclassifications (e.g. tagging a real bug as billing when both are valid enum values), prompt injection that successfully tricks the model into an incorrect label, or permanent network/API downtime.

## 5. One thing that broke, and how I found it
During our first `--eval` batch run, the first 4 tickets succeeded (100% match), but the remaining 8 failed with `Gemini API error 429` (quota exceeded) and `503` (high demand). 

By checking the error details and testing a single ticket individually after a pause, we found the cause: Google's free-tier has a limit of 5 requests per minute and 20 requests per day per model. Rapidly sending 12 tickets in a loop exceeded the rate limit. 

To fix it, we added an exponential backoff retry loop in `llm.py` that pauses when receiving 429/503 errors, and utilized `GEMINI_MODEL` to leverage separate model quotas when daily caps were reached.

## 6. Experiment 1: variation
One ticket (number 4), five runs each.

| Temperature | Labels across 5 runs | Different summaries | What changed |
|---|---|---|---|
| 1.0 | 5x account_access P2 | 5 | Vocabulary and sentence structure changed on every run while category and priority remained 100% stable. |
| 0 | 5x account_access P2 | 5 | Labels remained 100% consistent, and phrasing focused tightly on core lockout facts ("Customer cannot log in because OTP..."). |

## 7. Experiment 2: context
The one change I made to `SYSTEM_INSTRUCTION`: Added two labelled few-shot examples demonstrating edge cases: a payment link misreporting status (`bug` P2) vs a recurring invoice inquiry (`how_to` P4).

| | Category accuracy | Priority accuracy |
|---|---|---|
| Before | 12/12 (100%) | 12/12 (100%) |
| After | 12/12 (100%) | 12/12 (100%) |

What I think explains the difference: The baseline instruction was already well-defined; adding concrete few-shot examples reinforced boundary definitions on ambiguous tickets without introducing any regressions.
