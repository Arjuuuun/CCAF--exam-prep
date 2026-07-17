# CCA-F Domain 4: Prompt Engineering & Structured Output (20%)

*Note: Working through this domain via conceptual/API-based understanding, not Claude Code hands-on (no CLI access). Labs are replaced with API-equivalent framing where relevant.*

---

## Section 1: Prompt Engineering Fundamentals

### Core Concepts
1. **Clarity and directness beat cleverness.** Explicit instructions — what you want, the format, the constraints, the "why" — outperform vague prompts. Context helps Claude generalize correctly to edge cases.
2. **System prompt vs. user turn.** System prompt = persistent role/context/rules for the whole conversation. User messages = the specific task. Stable identity/constraints/tone belong in the system prompt — more consistently respected than mid-conversation instructions.
3. **Examples (multishot) are high-leverage.** 3-5 diverse examples, especially edge cases, often beat more instructional text for consistency. Bad examples reliably reproduce as bad behavior — Claude pattern-matches to *shown* behavior, not intent.
4. **Chain-of-thought / extended thinking** improves accuracy on multi-step, math, or logic-heavy tasks.
   - Prompted CoT: "think step by step" / `<thinking>` tags in a plain prompt.
   - Model-native extended thinking: a separate reasoning channel the model produces before the final answer, returned separately by the API.
5. **XML tags for structure.** Claude is trained to attend closely to XML-delimited sections (`<document>`, `<instructions>`, `<example>`, `<output_format>`). Tags reduce ambiguity about what's data vs. instruction vs. desired output.
6. **Prompt chaining.** Break a complex task into a pipeline of smaller single-objective prompts, one call's output feeding the next call's input. Trades latency/cost for reliability — each link is verifiable individually.
7. **Prefilling the response** (API-specific) — seed the start of the assistant turn (e.g., `{` for JSON) to constrain format or skip preamble.

### Key Terms
| Term | Definition |
|---|---|
| Multishot prompting | Providing several input/output examples in the prompt |
| Zero-shot / few-shot | No examples vs. a handful of examples given |
| System prompt | Top-level persistent instruction, separate from conversation turns |
| Prefill | Seeding the start of Claude's response text via the API |
| Prompt chaining | Sequential prompts where output of one feeds the next |
| Extended thinking | Native reasoning tokens produced before the final answer, budget-controlled |
| Temperature | Sampling randomness parameter (0 = deterministic-ish, higher = more varied) |

### Situation-Based Scenarios
**A — Inconsistent classification output.** Ticket classifier outputs vary in casing/wording. *Fix:* Add 4-5 multishot examples with exact label strings + enumerate the closed label set in a `<categories>` tag. Multishot anchors exact surface form; enumeration prevents invented categories.

**B — Long document + extraction task.** 50-page contract, need specific clauses. *Fix:* Put the document first in `<document>` tags, instructions after, ask Claude to quote the relevant sentence before answering. Long context before instruction improves recall; quote-first is cheap CoT-style accuracy boost.

**C — Multi-step task (analyze → summarize → draft email).** *Fix:* Chain three calls, each validated before feeding forward, instead of one giant prompt. Isolates failure points.

### Exam-Relevant Points
- Examples > instruction for consistent *format*; instructions > examples for conveying *reasoning/judgment logic*.
- XML tags materially improve parsing accuracy on structured/long prompts.
- Long reference material generally goes *before* the instruction/question.
- Prompt chaining = reliability technique, same family as "code beats prompt."

### Additional Context
Connects to **match-process-weight-to-risk** (one-shot prompt fine for low-stakes; chaining/examples/verification for high-stakes) and **narrow-scope-reliability** (one prompt, one job — same logic as narrow tool scoping).

### Practice Questions
1. You need Claude to always output category labels from a fixed list of 6 values with zero deviation in spelling/casing. What's the single most effective prompting addition?
2. True/False: Placing a 40-page reference document *after* your question in the prompt generally improves extraction accuracy.
3. What's the tradeoff prompt chaining makes to gain reliability?
4. In the API, what does "prefilling" the assistant turn let you control that a system prompt instruction alone often can't guarantee?
5. Why might bad examples in a multishot prompt be *worse* than no examples at all?

---

## Section 2: Precision Principles (Killing Ambiguity)

### Core Concepts
1. **Specific beats vague.** Vague instructions force guessing → inconsistent output across runs. ("Bring something nice" → "Bring 1kg Alphonso mangoes.")
2. **Categories beat feelings.** Subjective words ("important," "conservative," "high confidence") get interpreted differently each run. Replace with an explicit, enumerated rule set.
3. **One bad (noisy) category poisons trust in all categories** ("boy who cried wolf"). If one category in a multi-category task generates excessive false positives, disable/narrow it rather than let it erode confidence in categories that work.
4. **Give Claude an explicit list, not room to guess.** Enumerate exactly what to flag and what to ignore.
5. **Less can be more.** Asking for too many things at once degrades accuracy on all of them. Narrowing scope increases reliability on what remains (same idea as Domain 1/3's narrow-scope-reliability).
6. **Never use vague severity/quality labels — define them concretely.** "Critical," "high," "important" need explicit, concrete criteria (e.g., "high = hardcoded password") or classification varies run to run.

### Key Terms
| Term | Definition |
|---|---|
| Category-vs-feeling | Replacing subjective adjectives with enumerated, checkable categories |
| Scope narrowing | Deliberately reducing the number of things asked for in one pass to improve per-item accuracy |
| Severity anchoring | Defining what qualifies for each severity/label tier with concrete criteria, not adjectives |

### Situation-Based Scenario
Code review prompt asks for bugs, security, performance, style, and docs issues in one pass; style noise drowns out real bugs. *Fix:* Drop the noisiest category (style) from this pass; keep the 4 categories with strong signal; treat style as a separate later pass.

### Exam-Relevant Points
- Operationalizes Section 1's "specific beats vague" — adds the mechanism: subjective language → enumerated categories + concrete anchors.
- "Less can be more" = scope-management, same family as tool-scoping (~4-5 tools/agent) from Domain 3.

### Practice Questions
1. Why does telling Claude to "be conservative" produce inconsistent results across runs, and what's the fix?
2. In the "boy who cried wolf" analogy, what's the recommended action when one review category produces excessive false positives?
3. What's the difference between giving Claude a "list" vs. a "feeling" as an instruction?
4. Why does asking for 8 checks in one pass typically underperform asking for 4?
5. Give an example of a vague severity label and its concrete-anchor replacement.

---

## Section 3: Teaching by Example (Multishot Deep Dive)

### Core Concepts
1. **Examples beat instructions for consistency.** Describing a format in prose produces variable results; 1-2 input→output pairs lock in the exact structure every time.
2. **Examples resolve ambiguous/edge-case decisions.** Where instruction alone leaves a judgment call, examples encode the decision rule.
3. **Teach patterns, not isolated cases.** Vary examples across different instances of the same underlying pattern (div-by-zero, array out-of-bounds, missing file — all "runtime crash") so Claude generalizes the *category*, not the *instance*.
4. **Examples suppress hallucination.** Showing "unspecified" as a valid output for a missing value teaches Claude that "I don't know" is acceptable, rather than inventing plausible-looking data.
5. **Examples teach acceptable-pattern vs. genuine-issue distinctions**, reducing false positives.
6. **Cover format variation in your examples** so Claude generalizes across real-world input variation, not just the one format shown.
7. **2–4 examples is the sweet spot.** Too few → doesn't teach the pattern. Too many → wastes tokens without added benefit.
8. **Show the output format — don't describe it.** Demonstrating exact structure beats describing it in words.

### Key Terms
| Term | Definition |
|---|---|
| Pattern generalization | Using varied examples of the same underlying rule so Claude infers the category, not the literal case |
| Hallucination suppression via example | Explicitly demonstrating "unspecified"/"null" as valid output to prevent invented values |
| Format coverage | Including examples spanning the range of real-world input variation, not just one canonical form |

### Situation-Based Scenarios
**A — Field extraction hallucinating quantities.** Recipe source says "a pinch of salt," no quantity given. *Fix:* Example set includes a case where quantity is genuinely unspecified, expected output explicitly says `"unspecified"`. Result: honest missing-data reporting instead of invented numbers.

**B — Document format variation.** Citation extraction only trained on inline-citation examples; fails on bibliography-style docs. *Fix:* One example per format variant (bibliography, inline, numbered footnote) — 3 examples covering 3 known structures.

### Exam-Relevant Points
- Direct sequel to Section 1's multishot bullet — supplies the *why* (pattern-teaching, hallucination suppression) and the *tuning parameter* (2-4 examples).
- Reuses tracker meta-principle **don't-guess-surface-ambiguity** — examples are one concrete mechanism, alongside explicit categories (Section 2).

### Practice Questions
1. Why do varied examples of a "runtime crash" pattern generalize better than three examples of the exact same divide-by-zero case?
2. How do examples specifically reduce hallucination on missing/unspecified fields?
3. What's the recommended number of examples for teaching a pattern, and what happens outside that range on both ends?
4. If your extraction target document format varies, what should your example set look like?
5. Why is showing the exact output format via example generally more reliable than describing it in prose?

---

## Section 4: Structured Output — Tool-Use Enforcement over "Hoping for JSON"

### Core Concepts
1. **Tool use guarantees valid structure; asking for JSON in prose does not.** A tool's input schema forces syntactically valid, schema-conformant output — asking for "JSON with these fields" in plain text is a request, not a guarantee.
2. **`tool_choice` modes control enforcement strength:**
   - **`auto`** — Claude may or may not call a tool. Use when tool use is genuinely conditional.
   - **`any`** — Claude must call *some* tool, model picks which. Use when multiple schemas exist and the input type is unknown up front (e.g., extract-invoice vs. extract-resume vs. extract-contract).
   - **Forced/specific tool** — Claude must call *this exact* tool. Use when a pipeline requires a specific step, in a specific order, with no room to skip.
3. **Valid JSON ≠ correct data.** Schema validates *shape*, not *meaning*. Semantic checks (line items summing to total, realistic dates, cross-field consistency) must be added separately.
4. **Make optional fields genuinely optional (nullable), not required.** Forcing a "required" field genuinely absent from the source causes fabrication. Mark nullable, instruct Claude to use `null`/"not provided."
5. **Give enum fields an escape hatch** ("other," "unclear," "prefer not to say"). Without one, inputs that don't cleanly fit any category get force-fit, corrupting data.
6. **Normalize inconsistent source data via schema constraints** — exact date formats, integer amounts with stated currency, digits-only phone numbers — for downstream consistency.

### Key Terms
| Term | Definition |
|---|---|
| `tool_choice: auto` | Model may optionally call a tool or respond in plain text |
| `tool_choice: any` | Model must call one of the available tools, but picks which |
| `tool_choice: {tool: name}` (forced) | Model must call this specific tool |
| Syntax error (schema-level) | Malformed JSON, wrong types, missing required fields — caught by schema/tool validation |
| Semantic error (data-level) | Internally inconsistent or implausible values, still schema-valid — requires custom validation |
| Nullable field | An optional field explicitly allowed to be `null` rather than forced to always contain a value |
| Enum escape hatch | An "other/unclear" value added to a closed category set to avoid force-fitting |

### Situation-Based Scenarios
**A — Multi-document-type extraction, type unknown up front.** Define separate tools (`extract_invoice`, `extract_resume`, `extract_contract`), set `tool_choice: any`. Forces structured extraction via *some* schema while letting the model pick correctly — `auto` risks a non-extraction plain-text response; forcing one tool would be wrong for other document types.

**B — Multi-step pipeline where order matters.** Force `extract_metadata` as the required first tool call rather than `auto`, which could let Claude skip straight to a later step and break dependency order.

**C — Invoice extraction, schema valid but numbers don't add up.** Add a business-rule validation layer: sum of line items must equal stated total; flag mismatch. Schema only guarantees `total` is a number and `line_items` is correctly shaped — no concept of whether the numbers are *correct*.

### Exam-Relevant Points
- Scenario logic for `tool_choice`: conditional tool use → `auto`; multiple valid schemas, input type unknown → `any`; fixed pipeline order → forced.
- "Schema validates shape, not meaning" — pair with the line-item-sum-mismatch example.
- Nullable-field and enum-escape-hatch are both instances of: **don't force Claude to fabricate when the honest answer is "unknown/other."**

### Additional Context
Operationalizes Domain 3's `tool_choice` modes (`auto`/`any`/forced/`none`) — adds the decision framework for which mode to pick, plus extraction-specific patterns (nullable fields, enum escape hatches) not covered in Domain 3 notes. **Worth cross-referencing/merging with the Domain 3 file's `tool_choice` sub-section.**

### Practice Questions
1. You have three possible extraction schemas and don't know which document type you'll receive. Which `tool_choice` mode fits, and why not the other two?
2. An invoice extraction returns valid JSON with `total: 500` and line items summing to 450. What layer of validation catches this, and why doesn't tool-use schema enforcement catch it on its own?
3. Why does marking a field "required" when the source data sometimes lacks that value lead to bad outcomes, and what's the fix?
4. What's the purpose of adding an "other/unclear" value to an enum field?
5. In a forced multi-step pipeline, why would `tool_choice: auto` on the first step be risky?

---

## Section 5: Self-Correcting Extraction Systems

### Core Concepts
1. **Retry messages must state exactly what went wrong**, not just "try again." Generic retries reproduce the same mistake. Specific messages ("line item sum 450 ≠ invoice total 500, re-extract for consistency") let Claude self-correct.
2. **Retries cannot invent information that doesn't exist in the source.** If a required field is genuinely absent, retrying causes fabrication — wasting API calls, producing false data. Distinguish: *format errors* (fixable via retry) vs. *missing-source-information errors* (not retry-fixable — return null/unspecified instead).
3. **Track failure patterns, not individual failures**, to improve the underlying prompt/schema. Tagging findings by trigger condition (e.g., "f-string with hardcoded list → flagged as SQL injection") reveals systemic weaknesses that raw error counts don't.
4. **Two error types need two separate solutions:**
   - **Syntax errors** (malformed JSON, wrong types) → solved by tool-use/schema enforcement (Section 4).
   - **Semantic errors** (schema-valid but factually/logically wrong) → require separate custom validation layered on top.
5. **Build the self-correction loop with three inputs on retry:** original source document + the failed extraction + the specific validation error. Omitting any degrades correction quality.
6. **Know when to give up and stop retrying.** If failure is due to information genuinely absent from source, retrying is guaranteed to fail every time — classify the error type *before* deciding to retry.
7. **Track dismissed/rejected findings to improve prompts over time.** Tagging each finding with its triggering pattern lets you correlate "which trigger conditions are noisy" — untagged dismissals give no actionable signal.
8. **Design schemas that self-report their own inconsistencies** (e.g., `stated_total`, `calculated_total`, `conflict: true/false`) so the output flags its own problems rather than requiring independent re-verification.

### Key Terms
| Term | Definition |
|---|---|
| Format error | A retry-fixable error — info present in source but expressed incorrectly |
| Missing-source-information error | A non-retry-fixable error — the requested data doesn't exist in the source |
| Failure pattern tracking | Tagging findings/failures by triggering condition to identify systemic prompt weaknesses |
| Self-validating output | Schema design with redundant/derived fields plus an explicit conflict flag |

### Situation-Based Scenario
1000-document invoice extraction, 50 fail for various reasons (some missing required fields entirely, some malformed dates). *Fix:* Classify each failure by type before retrying — date-format failures get a targeted retry with the specific error appended; failures where the field is genuinely absent from source don't get retried (return null/unspecified). Blindly retrying all 50 wastes cost on the subset that can never succeed.

### Exam-Relevant Points
- Scenario logic: "should you retry this?" → is it a format issue (retry) or a source-absence issue (don't retry)?
- Connects to Domain 3's structured error handling "retryable" flag: format/syntax errors = retryable category; missing-source-info errors ≈ non-retryable.
- **Self-correction loop** (same instance, iterating with specific error feedback) is distinct from **fresh-eyes review** (Section 6 — separate instance/session, not retry).

### Practice Questions
1. Why does a generic "validation failed, try again" retry message tend to reproduce the same error?
2. Give an example of an error that should never be retried, and explain why retrying wastes money.
3. What are the three inputs a well-designed self-correction retry prompt should include?
4. Why is tracking failure patterns (tagged by trigger condition) more useful than just counting total failures?
5. What does it mean for a schema to be "self-validating," and give an example field design that achieves this?

---

## Section 6: Batch Processing Strategy

*Flag: this may sit partly in Domain 5 (Context Management & Reliability) — logged here since it's fundamentally about request/prompt architecture design. Reconcile placement once Domain 5 material is reviewed.*

### Core Concepts
1. **Batch API trades cost for latency**: roughly ~50% cheaper, no guaranteed fast turnaround (accept up to ~24hr delivery), no SLA guarantee on speed. Use only when the workload is latency-tolerant.
2. **Match the API to the workflow's blocking requirement:**
   - Sync API → anything actively waited on (pre-merge code review, user-facing chat).
   - Batch API → non-blocking, tolerant workloads (overnight reports, weekly audits, bulk translation/categorization/scoring).
3. **Batch API limitation: no multi-turn tool calling / back-and-forth flows.** Only single round-trip requests. Any workflow requiring "call tool → get result → reason further → call another tool" must use sync.
4. **Tag every batch request with a custom ID** to match individual responses back to their input — without this, a batch of parallel requests becomes unmatchable.
5. **Calculate batch submission frequency from your SLA**, not arbitrarily. A 4-hour SLA with once-daily submission breaks the SLA regardless of the 24hr batch ceiling — submit at a frequency that actually satisfies the requirement.
6. **On partial batch failure, resubmit only the failed records** (via custom ID), not the entire batch — resubmitting everything wastes cost on already-successful items.
7. **Test prompts on a small sample before running the full batch.** Refine against ~50 documents first; then scale to the full batch — dramatically cheaper than discovering a 30% failure rate at 10,000-document scale.

### Key Terms
| Term | Definition |
|---|---|
| Batch API | Asynchronous, cost-reduced (~50% off) processing mode with no fast-turnaround guarantee |
| Custom ID | A user-assigned identifier tagging each request in a batch so its response can be matched back |
| SLA-driven frequency | Calculating batch submission frequency based on actual business response-time requirement, not a default interval |
| Sample-first testing | Validating a prompt against a small subset before committing to full-batch cost |

### Situation-Based Scenarios
**A — Choosing sync vs. batch.** Pre-merge code review blocking a developer → sync (actively waited on). Nightly test-suite report → batch (non-blocking, cost savings apply).

**B — Partial batch failure.** 1000-document batch, 50 fail. Filter by custom ID to isolate the 50, apply targeted fixes per failure type (see Section 5), resubmit only those 50 — avoids doubling cost on the 950 that already succeeded.

**C — SLA of 4 hours, batch submitted once at midnight.** Increase submission frequency to match the SLA (e.g., every 4 hours) rather than relying on a single daily window, which could leave a document waiting 14+ hours before even entering the queue.

### Exam-Relevant Points
- Key disqualifier for batch: **any workflow requiring mid-request tool calls / multi-turn reasoning cannot use batch**, regardless of how non-urgent it seems. Likely trick-question axis.
- "Resubmit only failures" pairs with Section 5's error classification (retry-worthy vs. not) — expect these tested together.
- Approximate figures (~50% cost reduction, ~24hr ceiling) are sourced from a training-style summary, not verified against official docs — **treat as directionally correct, not exam-certain until verified.**

### Practice Questions
1. A workload requires searching docs, then analyzing results, then summarizing — all within one logical request. Can this run on the Batch API? Why or why not?
2. Why must every batch request be tagged with a custom ID?
3. Your SLA is 4 hours but your batch job runs once nightly. What's wrong with this design, and how do you fix it?
4. 1000 documents submitted, 50 fail. What's the cost-inefficient way to handle this, and what's the efficient way?
5. Why is testing a prompt on ~50 sample documents before running it on 10,000 a good cost-control practice?

---

## Section 7: Review Architecture — Fresh-Eyes Principle

*Flag: also possibly Domain 5-adjacent (reliability/verification architecture). Logged here since it came from the same source material batch; reconcile placement once Domain 5 content is reviewed.*

### Core Concepts
1. **The instance that generated output should not review it in the same session.** Same-session review carries the generation's reasoning context forward, so the model tends to defend/rationalize its own prior decisions rather than critically re-examine them.
2. **Fresh instances catch subtler bugs** — no "familiar blindness," no accumulated context biasing toward assuming correctness. Practically: generate in one session, review in a **new, separate session** with only the artifact (not the generation conversation) as input.
3. **Don't review everything at once — split into focused passes.** Two-pass structure:
   - **Pass 1 (per-file):** local bugs, file by file.
   - **Pass 2 (cross-file/integration):** consistency and interaction issues across files.
4. **Two instances, one truth: generator ≠ reviewer, always.** The structural rule underlying points 1-2.
5. **Attach confidence scores to findings** rather than treating all flagged issues as equally certain — enables smart routing (high-confidence → auto-action, low-confidence → human review).

### Key Terms
| Term | Definition |
|---|---|
| Same-session review bias | The tendency for a model to defend its own prior output when reviewing it within the same conversational context |
| Fresh-instance review | Reviewing generated output in a new session/instance with no access to the generation reasoning, only the artifact |
| Per-file vs. cross-file pass | Splitting review into a first pass over individual files (local issues) and a second pass across files (integration issues) |
| Confidence-scored finding | A flagged issue annotated with a certainty score to enable differentiated (auto-fix vs. human-review) handling |

### Situation-Based Scenario
Claude generates `auth.py`, then the same session is asked to "review the code you just wrote." *Fix:* Open a new session, paste in only `auth.py` (no generation history), ask for a bug/security review. The fresh instance has no prior commitment to correctness, evaluating it the way an independent reviewer would.

### Exam-Relevant Points
- Maps directly to the existing cross-domain meta-principle **independent/fresh-eyes review** (already logged from Domains 1-3) — this section supplies the concrete mechanism (new session, artifact-only input, no shared context).
- Likely trap-answer scenario: asking the same chat to review its own output. Correct answer requires a fresh session/instance.
- Confidence scoring pairs conceptually with Section 5's "classify before retry/act" pattern — both avoid treating all findings/errors as equally certain.

### Practice Questions
1. Why does reviewing generated code in the same chat session tend to produce a weaker review than a fresh session?
2. What exactly should be passed to the fresh review instance — the full generation conversation, or just the artifact? Why?
3. Why does reviewing 20 files in a single pass reduce review quality, and what two-pass structure addresses this?
4. What's the practical benefit of attaching a confidence score to a flagged finding, beyond just listing the finding?
5. State the "two instances, one truth" rule in your own words.

---

## Domain 4 Cheatsheet

| Principle | One-liner |
|---|---|
| Specific > vague | Give exact instructions, not adjectives — vague words force guessing |
| Categories > feelings | Enumerate rules instead of using subjective judgment words |
| One bad category poisons trust | Disable noisy categories rather than let them drown signal in good ones |
| Explicit lists > guesswork | Tell Claude exactly what to flag and what to skip |
| Less can be more | Narrow scope per pass improves accuracy on what remains |
| Concrete severity anchors | Define "high/critical/etc." with criteria, not adjectives |
| Examples > instructions | 2-4 examples lock in format/consistency better than prose description |
| Teach patterns, not cases | Vary examples across instances of the same rule to generalize |
| Examples stop hallucination | Show "unspecified" as valid output to prevent invented values |
| Tool use > hoping for JSON | Schema-bound tool calls guarantee valid structure; prose requests don't |
| `auto`/`any`/forced | Conditional → auto; unknown-schema-of-many → any; fixed pipeline step → forced |
| Valid JSON ≠ correct data | Schema checks shape; add custom checks for meaning/consistency |
| Nullable optional fields | Don't force required fields Claude can't always fill — allow null |
| Enum escape hatches | Add "other/unclear" to prevent force-fitting bad category matches |
| State exactly what went wrong | Retry messages need specific errors, not just "try again" |
| Retries can't invent facts | Don't retry missing-source-info errors — retry only format errors |
| Classify before retry | Syntax error (retry) vs. semantic/missing-info error (may not be retryable) |
| Track patterns, not counts | Tag failures/dismissals by trigger condition to fix root causes |
| Self-validating schemas | Include derived fields + conflict flags so output flags its own problems |
| Batch = cheaper, slower | ~50% cost savings, no fast-turnaround guarantee — only for tolerant workloads |
| Batch can't multi-turn | No mid-request tool calls — single round-trip only |
| Custom IDs in batch | Tag every request to match responses back to inputs |
| SLA drives batch frequency | Submission cadence must satisfy actual response-time requirement |
| Resubmit failures only | Don't reprocess already-successful batch items |
| Sample before scaling | Test/refine on ~50 docs before running the full batch |
| Fresh-eyes review | Generator ≠ reviewer — use a new session with artifact-only input |
| Per-file then cross-file | Split review into focused passes rather than reviewing everything at once |
| Confidence-scored findings | Attach certainty to findings to enable auto-fix vs. human-review routing |

---

## Gaps Tracker (carried forward + new)
- `/compact` command (referenced, never covered)
- MCP prompts primitive (tools ✅, resources ✅, prompts ❌)
- Exit-code conventions for CI/CD
- User-level's exact position in CLAUDE.md 3-way precedence (inferred, not confirmed)
- Exact `Edit` tool parameter schema
- **Batch API exact figures** (50% discount, 24hr ceiling) — sourced from informal material, not official docs; verify before treating as exam-certain
- **Domain placement ambiguity** — Sections 6 (Batch Processing) and 7 (Review Architecture) may belong partly/wholly under Domain 5; reconcile once Domain 5 material is reviewed
- `tool_choice: none` mode — referenced in Domain 3 recap but not detailed in this material; still open
