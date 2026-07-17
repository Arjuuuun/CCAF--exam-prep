# CCA-F Domain 5: Context Management & Reliability (15%)

---

## Section 1: Context Window Management

### Core Concepts
1. **Facts beat summaries.** A summary discards the specific, checkable details (quantities, IDs, dates, amounts) that make information actionable and verifiable later. Always prefer passing concrete facts over paraphrased summaries when that data will be needed downstream. ("50kg rice, ₹62/kg, delivered Nov 15" survives; "some rice order pending" doesn't.)
2. **The middle of a long context gets lost ("lost in the middle").** Models attend most reliably to the beginning and end of a long context; content buried in the middle is more likely to be skipped or under-weighted — same effect as a reader remembering a book's intro/conclusion but not chapters 7-12. **Mitigation:** put critical information at the top (or explicitly flagged/headed), don't rely on the model to surface it from the middle unprompted.
3. **Tool results eat your context window — trim/filter at the source.** Returning full, wide tool outputs (e.g., 40 fields when only 5 are needed downstream) wastes context budget and dilutes attention on what's actually needed. Filter fields at the function/tool level *before* the output enters the model's context, not after.
4. **Claude (the API) has no memory — every call is stateless.** Each API call is independent; the model has no built-in continuity between calls. To "continue a conversation," the full conversation history must be passed in on every single call. This is the mechanical foundation of how chat applications are built on top of the base model — there's no server-side session state on the model's end unless the calling application constructs and re-sends the history.
5. **Pin the numbers — don't let facts get paraphrased away in narrative summaries.** A narrative summary ("patient had high BP, was on medication") loses the specific, actionable numbers ("BP 150/95, on [medication], visited [date]"). Where numbers/IDs/dates matter downstream, keep them as explicit structured facts alongside (or instead of) narrative summary.
6. **Structure beats prose for multi-issue sessions.** When a single session/context contains multiple distinct issues or tickets, blending them into flowing prose causes cross-contamination — details from one issue bleed into or get confused with another. Structuring each issue into its own explicit block prevents this crossover.
7. **Headers save the middle.** Within long documents/contexts, using headers, bolded lead summaries, and subheadings for critical sections pulls attention to those sections regardless of their position — directly counteracting the lost-in-the-middle effect. A leading summary block plus tagged sections ensures no critical fact is missed even in a hurried pass.
8. **Sub-agents must pack their receipts (cite sources with their outputs).** When a sub-agent (or any component) returns a finding or recommendation, it should include the supporting evidence — source, date, specific figures — not just a bare conclusion. "Vendor A seems best" is unverifiable; "Vendor A, ₹4.2 lakh, quoted Nov 2, sourced from [email]" is trustable and auditable.

### Key Terms
| Term | Definition |
|---|---|
| Lost-in-the-middle effect | The tendency for content in the middle of a long context to receive less attention than content at the start or end |
| Context trimming / filtering at source | Reducing tool/function output to only the fields actually needed before it enters the model's context |
| Statelessness | Each API call is independent with no server-side memory; full history must be re-sent every call to maintain continuity |
| Fact-pinning | Preserving specific, structured data (numbers, IDs, dates) rather than letting it dissolve into narrative paraphrase |
| Structured multi-issue blocking | Giving each distinct issue/ticket its own explicit block in the context rather than blending them in prose |
| Receipt-packing | A sub-agent or component including its supporting evidence/source alongside its conclusion |

### Situation-Based Scenarios

**A — Long multi-report summarization missing the critical one.**
*Input:* 50 reports total, 8 flagged as critical (customer churn risk), asked to "just summarize."
*Mechanism:* Instead of a flat summarize-everything instruction, explicitly instruct: lead with the churn-risk findings first, then proceed to the individual reports.
*Why it works:* Prevents the critical subset from being buried in the middle of an undifferentiated 50-report pass.

**B — Tool call returning bloated records.**
*Input:* An order-lookup tool returns 40+ fields (order ID, customer ID, billing address, shipping address, etc.) per call, and only 5 fields are needed for the next step.
*Mechanism:* Configure the tool/function to return only the needed fields (order ID, status, amount, delivery date, etc.) at the source, rather than returning everything and letting the model ignore the rest.
*Why it works:* A full 40-field return across 10 calls can meaningfully consume the context window and dilute attention even if the unused fields are technically "ignorable."

**C — Multi-turn conversation losing coherence.**
*Input:* A customer support chat is expected to "remember" what was decided earlier without re-sending it.
*Mechanism:* Every API call must include the complete conversation history (all prior user/assistant turns), not just the latest message.
*Why it works:* The model has no memory between calls; omitting history means the model has no way to know what was previously discussed or decided.

**D — Multi-ticket session with crossed details.**
*Input:* One session contains a login issue, a ₹500 refund request, and a billing question, described in flowing prose.
*Mechanism:* Structure the context into explicit per-issue blocks (Issue 1: ..., Issue 2: ..., Issue 3: ...) rather than one blended narrative.
*Why it works:* Prevents the model from conflating details across unrelated issues.

### Exam-Relevant Points
- Expect a scenario question testing whether you'd trim tool output vs. pass it through raw — the correct answer is almost always filter at the source, matching the earlier meta-principle of narrow scope for reliability.
- Statelessness is a likely explicit exam point: "why does a chat application need to resend the full conversation on every call?" → because Claude/the API has no session memory.
- Lost-in-the-middle + headers-save-the-middle should be paired: the problem (middle content under-attended) and the fix (headers/leading summaries/explicit flagging) are two sides of the same exam concept.
- "Facts beat summaries" and "pin the numbers" are the same underlying principle applied at two different scales (a whole record vs. specific numeric fields) — don't treat them as fully separate ideas if a question conflates them.

### Additional Context
This section is the direct mechanical explanation behind several things you already logged as assumed/black-box in earlier domains:
- **Domain 1's session management** (name/resume/fork/fresh) only works *because* the calling application/tooling reconstructs and resends history — this section explains *why* that's necessary (statelessness).
- **Domain 1/3's narrow-scope-reliability** meta-principle gets a concrete mechanism here: trimming tool output fields at the source is a direct application of "narrow scope = higher reliability," specifically for context window management rather than tool selection.
- **Sub-agents must pack their receipts** connects to Domain 1's sub-agent spawning principles (isolated-execution-with-summary-return) — this adds the requirement that the *summary itself* must carry sourced, verifiable facts, not just a bare conclusion.

### Practice Questions
1. Why does summarizing a customer complaint into prose narrative risk losing information that a later step (e.g., processing a refund) will need?
2. What is the "lost in the middle" effect, and name two concrete mitigations covered in this section.
3. A tool call returns 40 fields when only 5 are needed downstream. What's the recommended fix, and why not just let the model ignore the unused fields?
4. Why must a chat application resend the entire conversation history on every single API call?
5. You have one session covering three unrelated customer issues. What structural approach prevents the model from confusing details across them?
6. What does it mean for a sub-agent's finding to "pack its receipts," and why does this matter for trustworthiness of the output?

---

## Domain 5 Cheatsheet (running)

| Principle | One-liner |
|---|---|
| Facts beat summaries | Preserve specific figures/IDs/dates rather than paraphrasing them away |
| Lost in the middle | Content in the middle of long context gets under-attended — front-load or flag critical info |
| Trim tool output at source | Return only the fields needed downstream, not the full raw record |
| Statelessness | Every API call is independent — full conversation history must be resent each time |
| Pin the numbers | Keep specific numeric/ID data structured, not dissolved into narrative |
| Structure > prose for multi-issue | Block each distinct issue separately to prevent cross-contamination |
| Headers save the middle | Leading summaries + tagged sections counteract the lost-in-the-middle effect |
| Sub-agents pack receipts | Findings must include sourced evidence, not bare conclusions |

---

## Gaps Tracker (carried forward from Domain 4)
- `/compact` command (referenced, never covered) — **likely belongs in this domain; watch for coverage in later Domain 5 sections**
- MCP prompts primitive (tools ✅, resources ✅, prompts ❌)
- Exit-code conventions for CI/CD
- User-level's exact position in CLAUDE.md 3-way precedence (inferred, not confirmed)
- Exact `Edit` tool parameter schema
- Batch API exact figures (50% discount, 24hr ceiling) — sourced from informal material, not official docs; verify before treating as exam-certain
- `tool_choice: none` mode — referenced in Domain 3 recap but not detailed in any material yet

### Domain placement reconciliation (from Domain 4 notes)
Now that Domain 5 content is starting, tentative call:
- **Domain 4 Section 6 (Batch Processing)** — stays in Domain 4. It's fundamentally about request/API architecture choice (sync vs. batch), not context window or reliability management. No overlap with this section's content.
- **Domain 4 Section 7 (Review Architecture / fresh-eyes)** — this is a **reliability** technique (independent verification, error-catching), which fits Domain 5's naming ("Context Management & **Reliability**") better than Domain 4. **Recommend moving/duplicating a reference to it here once more Domain 5 material confirms the pattern** — holding off on physically moving it until you've seen more of Domain 5's scope, in case it's addressed again with more depth under this domain.
