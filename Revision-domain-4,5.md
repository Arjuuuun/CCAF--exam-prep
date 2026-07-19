# CCA-F Quick-Read: Domains 4 & 5 (Pre-Mock Refresher)

## Domain 4 — Prompt Engineering & Structured Output Precision

- Specific > vague. Categories > feelings.
- One noisy category kills trust in all — disable it, don't average it in.
- Vague severity labels ("high/critical") need concrete anchors, not adjectives.
- Less is more — narrow scope per pass beats asking for everything at once.

### Examples

- Examples > instructions for format consistency.
- 2–4 examples = sweet spot.
- Vary examples to teach the pattern, not one instance.
- Show "unspecified"/null in examples to stop hallucination on missing data.
- Show the exact format — don't describe it.

### Structured Output

- Tool use (schema-enforced) > hoping for JSON in prose.
- `tool_choice`:
  - **auto** = conditional/optional
  - **any** = must call one of several schemas, unknown input type
  - **forced** = must call this tool, fixed pipeline order
- Schema validates shape, not meaning — add custom checks for cross-field consistency (totals summing, realistic dates).
- Required fields Claude can't always fill → make nullable, don't force fabrication.
- Enum fields need an "other/unclear" escape hatch.

### Self-Correction

- Retry messages must state exactly what was wrong — not "try again."
- Retries can't invent missing source data — classify format error (retry) vs missing-info error (don't retry, return null) before retrying.
- Self-correction loop needs 3 things: original doc + failed output + specific error.
- Track patterns of failure/dismissal, not raw counts, to fix root cause.
- Self-validating schema = include derived fields + a conflict flag.

### Batch Processing

- Batch = ~50% cheaper, no fast-turnaround guarantee. Only for latency-tolerant work.
- Batch cannot do multi-turn tool calls — single round-trip only.
  - (Common trick: "seems non-urgent" ≠ automatically batch-eligible if it needs multi-turn tools.)
- Tag every request with a custom ID.
- Resubmit only failures, not the whole batch.
- Submission frequency should be driven by SLA, not a default daily window.
- Test on a small sample before scaling to the full batch.

### Review Architecture (fresh-eyes)

- Generator ≠ reviewer, ever, in the same session — same-session review defends its own decisions.
- Fresh review = new session, artifact only, no generation history.
- Split review into per-file pass then cross-file/integration pass — don't review everything at once.
- Attach confidence scores to findings to route auto-fix vs. human review.

---

## Domain 5 — Context Management & Reliability

### Context Window

- Facts > summaries — pin numbers/IDs/dates, don't paraphrase them away.
- Lost in the middle: content in the middle of long context gets under-attended.
  - Fix: headers, leading summaries, front-load critical info.
- Trim tool output at the source — return only needed fields, not everything.
- Statelessness: every API call is independent — full history must be resent every time.
- Multi-issue sessions: structure into separate blocks, don't blend in prose.
- Sub-agents must pack receipts (source, date, figures) with findings — not bare conclusions.

### Escalation Design

- Only 3 valid triggers: explicit human request, policy gap, no progress.
  - "Seems complex" is **NOT** a trigger.
- Explicit human request → transfer immediately, no "let me check first."
- Offer resolution first for straightforward issues; escalate only if reiterated.
- Never escalate on sentiment or self-reported confidence — not objective.
- Multiple tool matches → ask for a disambiguating identifier, never guess by heuristic (recency, similarity).
- 2–4 concrete examples in system prompt teach escalation behavior better than an abstract rule.
- Policy silent on a specific ask → escalate, don't extrapolate/invent coverage.

### Error Propagation

- Errors need context (what + why), not just a fail flag.
- Empty result ≠ broken — "0 matches, search completed" is a valid success, not a failure.
- Generic errors block recovery; specific errors enable fixing.
- Neither silent nor catastrophic — surface visibly, keep going where safe.
- Try locally first (retry a few times) → escalate only if still failing.
- Annotate coverage gaps explicitly (which regions/sources have data) — don't generalize from partial data.

### Large Codebase Exploration

- Context decay: long sessions drift from grounded facts to vague generic assumptions.
  - Vague answer = decay already happened.
- Scratchpad the truth: write findings to a persistent file, don't trust in-context memory.
- Delegate the verbose: split large file-reading across sub-agents by category, each returns a summary.
- Summarize before spawning: give sub-agents a specific, detailed brief — not "investigate further."
- Manifest survive crisis: persist task state so a crash doesn't mean starting over.
- Compact when cluttered: periodically strip noise, keep current findings only.
- Coordinate, not inspect: main agent directs + synthesizes; doesn't personally read everything.

### Human Review & Confidence

- Averages lie — a good aggregate can hide a bad subcategory. Always slice by segment.
- Validate before automate — measure real accuracy on a sample; don't automate on "looks correct."
- Sample the confident — spot-check even high-confidence output, don't trust blindly.
- Confidence ≠ accuracy — a confidence score is only useful if calibrated against real outcomes.
- Route the doubtful only to human review — not every case (wastes expert time/creates bottlenecks).

### Provenance

- Sources don't survive by default — attach a source to every individual claim, not one blanket citation.
- Annotate conflicts — show disagreeing sources side by side, never silently pick one.
- Date every claim — a number without a timeframe can look falsely contradictory.
- Separate settled vs. contested claims explicitly.
- Format follows content — don't blend different info types into one undifferentiated block.

---

## ⚠️ Newly Identified Gaps (from mock exam review — not fully in original notes)

- **Tool-reach restriction, not just tool-count scoping.** If a tool (e.g., URL fetch) can wander outside an approved source set, restrict what it can access (domain/catalog allow-list, code-enforced) — not just how many tools an agent has.
- **Route by task-predictability type.** If one agent handles both a stable-rubric task (checklist, known steps) and a variable/exploratory task (investigation, unknown path) in a single prompt, split into two distinct workflows — enforce the rubric one structurally (code/deterministic), leave the exploratory one agentic. Don't force one single-pass prompt to do both.
- **Static knowledge belongs in context, not behind a tool call.** If an agent repeatedly burns turns re-querying the same stable, non-customer-specific info (general policies) via a tool, that's a signal to preload it into system prompt/context instead — reserve tool calls for genuinely dynamic, customer-specific lookups.

> **Also remember:** when a question combines two principles (e.g., "sub-agents need receipts" + "structured output beats prose"), the answer is usually the combined enforcement — structured, schema-required citation fields, not just "tell the sub-agent to cite sources."
