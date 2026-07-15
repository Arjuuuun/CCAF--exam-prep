# CCA-F Domain 1: Agentic Architecture & Orchestration
## Consolidated Study Notes (27% of exam — highest weighted domain)

---

## 1. The Agent Loop & `stop_reason`

**Underlying concept:** An agentic loop is a `while` loop your server code runs against the raw Claude API. Claude has no built-in "loop" — your application code calls the API, inspects the response, decides whether to act again, and calls again. The `stop_reason` field is the *only* reliable signal for when to continue vs. stop.

**Input to the API call:** messages array (conversation history so far) + `tools` parameter (list of available tool schemas) + system prompt.

**Key output parameter checked:** `response.stop_reason` — values: `tool_use` (continue loop, execute tool), `end_turn` (break loop, return final answer), also `max_tokens` and `stop_sequence` (edge cases — not covered in transcript but real API values).

**How it's done (mechanism):**
1. Send messages + tools to `/v1/messages`.
2. Check `stop_reason`.
3. If `tool_use` → execute the matching tool in your own code → append tool result to messages array → call API again.
4. If `end_turn` → return Claude's text as final answer, exit loop.

**Situation-based scenario:**
> *You're building a support agent. A user asks "what's my order status?" without providing a customer ID. Claude responds asking for the ID — no tool call. Later, a user provides both customer ID and order ID; Claude calls a `get_order` tool, gets data back, then responds with the answer.*
> - First case: `stop_reason = end_turn` immediately (Claude decided it had enough to just ask a clarifying question — correct behavior, no guesswork needed from your code).
> - Second case: `stop_reason = tool_use` → your code runs `get_order()` → appends result → calls Claude again → second response has `stop_reason = end_turn`.

**Anti-pattern (exam trap):** Stopping the loop when generated text contains words like "done," "finished," or when text is merely non-empty. This is unreliable — the model isn't guaranteed to use any specific word, and text-based stopping can terminate a genuinely unfinished task or run forever on a task that never says the "magic word."

---

## 2. Coordinator & Sub-Agent — 5 Rules

**Underlying concept:** Multi-agent systems need a topology. Hub-and-spoke (coordinator-centric) keeps the system observable and controllable versus peer-to-peer chains where agents call each other directly and errors become untraceable.

| Rule | Input/Parameter Involved | Mechanism | Situation |
|---|---|---|---|
| **One hub, many spokes** | Coordinator's tool list includes each specialized sub-agent as a callable "tool" | All sub-agent calls and results route back through the coordinator; sub-agents never call each other | *Search agent finds data → should report to coordinator, not call the Writer agent directly. Coordinator then decides to invoke Writer with that data.* |
| **Sub-agent starts fresh** | The **prompt/task description** passed at spawn time is the sub-agent's *entire* context | Sub-agent has no access to coordinator's chat history; only what's explicitly packed into its spawn prompt | *Coordinator has 50 messages of research history. Spawning a new search agent with just "find more" fails — must pass topic + what's already found + what's still needed.* |
| **Coordinator picks the right helpers** | Coordinator reads the query first, then selects only relevant sub-agents from its available set | Conditional/selective delegation logic — not "run every sub-agent every time" | *Simple factual question → only 1 sub-agent needed, not the full search→analyze→write pipeline.* |
| **Split without gaps** | Task decomposition defines each sub-agent's scope boundary | Each sub-agent gets a non-overlapping slice of the problem; coordinator ensures full coverage | *3 agents on "EV battery market": pricing / recycling / policy — not two agents both covering pricing while recycling goes uncovered.* |
| **Check, refill, repeat** | Output of pass 1 becomes input to a "find gaps" step, then a "fill gaps" step | Coordinator doesn't finalize on first pass — iterates: generate → detect gaps → dispatch follow-up sub-agent task → repeat | *Summary report has a missing section → coordinator spawns a "fill gap: recycling costs" sub-agent before finalizing.* |

**Situation-based scenario (combined):**
> *Building a research coordinator for "quarterly market analysis." A badly designed version has Agent1→Agent2→Agent3 calling each other directly with no logging. When Agent2 fails, no one knows why. Correct design: Coordinator spawns Agent1 (pricing), Agent2 (competitors) in parallel — both report back to coordinator only — coordinator reviews combined output, finds a gap in "regulatory risk," spawns Agent3 with that specific gap as its task, then synthesizes the final report.*

---

## 3. Spawning Sub-Agents — 7 Principles

| Principle | Parameter/Config | Mechanism | Situation |
|---|---|---|---|
| **Open the task door** | A "task tool" must be defined and included in the coordinator's `tools` list | Without this tool schema present, the coordinator literally cannot delegate — it's a permission/capability gate, not a prompt instruction | *Coordinator's tools = [search, write]. No task tool → coordinator cannot spawn any sub-agent no matter how the prompt is worded.* |
| **Pack the sub-agent's bag** | Spawn-time prompt includes topic, prior findings, target goal | Sub-agent begins work immediately using only what's passed at creation | *Spawning a "summarize" agent with zero data vs. spawning it with the actual document/topic to summarize.* |
| **Define each agent's role** | Each sub-agent config specifies role + its own scoped tool subset | Clear, bounded role prevents ambiguous/chaotic behavior | *"Research agent" with access only to a `search` tool — vs. an undefined agent expected to do research, writing, and analysis all at once.* |
| **Fork to explore options** | Shared "base" output is computed once, then passed to N divergent branches | Avoids redoing common analysis per option — do base work once, branch after | *Comparing 2 pricing strategies: do market analysis once → fork into Strategy-A agent and Strategy-B agent, each building only on top of the shared base.* |
| **Keep sources attached** | Each fact/finding is tagged with its origin (doc name, report, page) | Enables downstream verification and traceability | *"Sales grew 20%" → must be paired with "Source: Q2 report," not stated as a bare fact.* |
| **Pound in parallel** | Independent sub-agent tasks are launched concurrently (async/parallel calls) rather than sequentially | Reduces total latency when tasks don't depend on each other's output | *Looking up order 5001 and order 5003 for the same customer — no dependency → launch both sub-agent calls at once.* |
| **Set goals, not steps** | Task description specifies success criteria/goal, not a fixed step sequence | Lets the sub-agent adapt its own approach if a step fails or conditions change | *"Produce an AI market report using 3+ trusted sources" (goal) instead of "search → read paper → summarize" (rigid steps that break if search returns nothing).* |

**Situation-based scenario:**
> *You're asked to build an agent that compares pricing across 3 vendors. Bad design: sequentially spawn Vendor-A analysis from scratch, then Vendor-B from scratch, then Vendor-C from scratch — each redoing the same "what data do we need" groundwork, one after another. Good design: do the shared groundwork (what fields matter, what data source to check) once, fork into 3 parallel sub-agents (goal: "find current pricing for Vendor X using the verified data source, cite it"), run all 3 concurrently, each returns a sourced finding.*

---

## 4. Workflow Enforcement & Clean Hand-offs — 4 Rules

**Underlying concept:** Prompt instructions are *probabilistic* (the model might skip them); critical business rules need to be *deterministic*, meaning enforced in code, not just requested in a prompt.

| Rule | Parameter | Mechanism | Situation |
|---|---|---|---|
| **Code beats prompt** | Validation logic lives in the tool's implementation code | The tool itself checks a condition and refuses to execute if unmet — model cannot bypass this | *Prompt says "verify customer before refunding" — model occasionally skips it. Fix: the `process_refund()` function itself checks a `verified` flag and throws/returns an error if false.* |
| **Guard the critical step** | Prerequisite condition check before high-stakes tool execution | Tool refuses to run until a required prior condition is satisfied (analogous to OTP-before-transfer) | *Bank transfer tool requires OTP verification flag = true before executing — no code path allows transfer without it.* |
| **One request, many concerns → decompose + parallelize** | Coordinator splits a bundled request into independent sub-tasks | Multiple concerns handled together, run in parallel when independent | *"Fix my car" = oil change + brakes + AC → 3 sub-tasks dispatched in parallel instead of solving only one and dropping the rest.* |
| **Clean hand-off** | Escalation/delegation payload includes structured specifics, not a vague pointer | Receiving agent/human can act immediately without re-deriving context | *Escalating to a human: "duplicate payment, customer 1001, order 5002, $340, already refunded once on July 10" — not just "billing issue, please help."* |

**Situation-based scenario:**
> *An agent handles refund requests. Without code enforcement, it sometimes processes a refund without checking identity because the prompt instruction got "lost" in a long context. Fix: move the identity check into the refund tool's own code — `if not verified: return {"error": "verification required"}`. Additionally, if a refund exceeds a threshold, instead of just blocking it outright, the tool routes it to an "escalate_to_human" tool with full context (amount, customer, reason) attached — a clean hand-off, not a dead-end rejection.*

---

## 5. Hooks for Interception & Data Normalization — 4 Principles

**Underlying concept:** Hooks are code-level intercepts around tool execution, tied to a lifecycle event. They are **guaranteed** to run (they're just code execution, not something the model can skip), unlike prompt instructions which are probabilistic.

| Principle | Hook Type | Parameter | Mechanism | Situation |
|---|---|---|---|---|
| **Clean before reading** | **Post-tool-use hook** | Raw tool output | Runs *after* tool executes, *before* result reaches the model — standardizes format (e.g., date formats) | *A `get_order` tool returns dates in 3 different formats depending on source system — post-tool hook normalizes all to ISO 8601 before Claude ever sees them.* |
| **Catch the call mid-air** | **Pre-tool-use hook** | Proposed tool call + its arguments | Runs *before* tool executes — can block the call entirely if a rule is violated | *Refund tool call proposes `amount=1000` but policy limit is 500 — pre-tool hook intercepts and blocks before the refund tool ever runs.* |
| **Block then redirect** | Pre-tool-use hook (extended) | Same as above, plus a fallback action | Instead of simply blocking, hook redirects the flow to an alternate valid path (e.g., escalation) | *Refund > 500 → pre-tool hook blocks the refund tool call AND triggers an escalate-to-human tool instead, rather than just returning an error with no next step.* |
| **Hooks hold the line** | Either | N/A — comparative principle | Hooks are guaranteed (shell/code execution outside model control); prompts are probabilistic | *"Get approval before deleting a user" as a prompt instruction can occasionally be skipped by the model. As a hook: `if not approved: block()` — cannot be skipped, ever.* |

**Situation-based scenario:**
> *A delete-user agent occasionally deletes without approval despite a clear prompt instruction. Diagnosis: the rule lives only in the prompt (probabilistic). Fix: implement a pre-tool-use hook on the `delete_user` tool that checks an `approved` flag and returns a block/error if false — this makes the rule un-skippable regardless of what the model "decides." Separately, a post-tool-use hook on a `get_customer_data` tool normalizes inconsistent phone number formats from different backend systems before the data reaches Claude.*

**Real mechanism note (beyond transcript):** In Claude Code, hooks execute as shell commands tied to specific lifecycle events (not just pre/post-tool-use) — e.g., `SessionStart`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `PreCompact`, `Notification`. Know that hooks operate **outside model control entirely** — this is the literal reason they're deterministic.

---

## 6. Task Decomposition — Matching Strategy to Workflow

**Underlying concept:** This restates the "agents vs. workflows" distinction at the planning level. Known, predictable tasks → fixed chain. Unknown/discoverable tasks → adaptive plan. This is a **planning-strategy** decision, separate from delegation mechanics (Sections 2–3).

| Rule | Parameter | Mechanism | Situation |
|---|---|---|---|
| **Match strategy to workflow type** | Task predictability (known vs. unknown path) | Choose chain (fixed) or adaptive plan (flexible) based on task type | *Order fulfillment (known steps) → chain. Open-ended debugging (unknown cause) → adaptive plan.* |
| **Chain when path is known** | Fixed ordered step list | Execute steps strictly in sequence | *Restaurant flow: take order → cook → serve → bill — always in that order.* |
| **Per-file first, then cross-file last** | Two-pass review: local pass + integration pass | Evaluate each unit individually first, then a separate pass for cross-unit issues | *Code review: review file A alone, review file B alone, THEN a separate pass checking how A and B interact — not one combined pass trying to catch both types of issues at once (causes attention dilution / missed or contradictory findings).* |
| **Adapt as you discover** | Initial plan + revision loop based on new findings | Plan is revised at each step as new information emerges, not fixed upfront | *"Something's wrong with my order, sort it out" (no order ID given) → list all orders first → THEN decide which to investigate based on what's found → THEN check applicable policy based on what's discovered.* |

**Situation-based scenario:**
> *A code-review agent is given 5 files to review plus their interactions. Bad design: one single pass trying to catch both local bugs and cross-file integration issues simultaneously — findings become inconsistent or incomplete. Good design: Pass 1 reviews each file independently for local issues; Pass 2 (separate) specifically checks data flow and contradictions across files. Separately, a debugging agent given a vague "something's broken" ticket cannot use a fixed chain — it must start with an exploratory step (list recent changes/errors), then adapt its plan based on what that reveals.*

**Exam nuance:** Real systems are often **hybrid** — outer structure can be a fixed chain (e.g., always: gather → analyze → report) while one stage internally uses adaptive planning.

---

## 7. Session Management — Resume, Fork, Fresh

**Underlying concept:** Managing conversation state/context over time — Claude Code–specific in practice (native session resume/fork features), though the same principles apply if hand-built in a raw API application (as shown in the hands-on lab, using local JSON files).

| Rule | Parameter | Mechanism | Situation |
|---|---|---|---|
| **Name it, then resume it** | Session identifier (name/ID) | Persisted history keyed by name; reloading by name restores full prior context | *Session "auth-bug" resumed a week later picks up exactly where it left off, no re-explaining.* |
| **Fork to compare paths** | Shared session state at fork point, cloned into a new session ID | Common/base context copied once; each fork diverges independently from there | *"asa-case" session forked into "asa-alt" — both start identical, then diverge as separate conversations explore different resolutions.* |
| **Tell the agent what changed (targeted resume)** | Explicit diff/changelog passed at resume time | Agent re-analyzes only the stated delta, not the full prior state | *Resuming a code-review session: "only auth.py changed" → agent reviews just that file, not the whole codebase again.* |
| **Start fresh when stale** | New session seeded with a short summary instead of full raw history | Prevents reasoning from outdated facts; also reduces context bloat (manual context compaction) | *A week-old session referencing since-changed pricing data → start a new session injected with a fresh summary of current state, not the old raw transcript.* |

**Situation-based scenario:**
> *A support agent session named "asa-case" has established that the user is asking about order 5001. User quits and returns later — resuming "asa-case" lets them ask "is that refundable?" and the agent correctly resolves "that" to order 5001 from history. Later, the user wants to explore an alternate resolution path without losing the original — forking into "asa-alt" preserves both. Eventually the session grows long and partially outdated — starting fresh with an injected one-paragraph summary avoids both staleness and unnecessary context bloat.*

**Key distinction for exam:** Stateless raw API (you manage history yourself, e.g., JSON files as shown in the lab) vs. Claude Code's native session resume — both implement the same principle at different architectural layers.

---

# 🎯 EXAM-DAY CHEATSHEET

### The One Recurring Meta-Principle (tested repeatedly across framings)
**Deterministic code > probabilistic prompts**, for anything that must never fail:
- Agent loop control → use `stop_reason`, not text-matching
- Business rules → enforce in tool code / hooks, not just prompt wording
- Critical actions → gate with pre-tool-use hooks or tool-level validation, not instructions alone

### `stop_reason` Quick Reference
| Value | Meaning | Loop Action |
|---|---|---|
| `tool_use` | Claude wants to call a tool | Execute tool → append result → call again |
| `end_turn` | Claude is done | Break loop, return answer |
| `max_tokens` | Hit token limit | Retry with more budget (edge case) |
| `stop_sequence` | Hit custom stop string | Edge case handling |

### Multi-Agent Design Checklist
- [ ] Hub-and-spoke topology (no direct sub-agent-to-sub-agent calls)
- [ ] Task tool enabled on coordinator (delegation requires this explicitly)
- [ ] Sub-agent spawn prompt includes full context (topic, prior findings, goal) — sub-agents start with zero memory
- [ ] Sub-agent given a clear, bounded role + scoped tool access
- [ ] Independent tasks → parallel; dependent tasks → sequential with explicit context passing
- [ ] Shared/common work computed once, then forked for divergent branches
- [ ] Facts carry source attribution
- [ ] Coordinator does gap-check-and-refill passes, not single-shot finalization
- [ ] Sub-agents given goals/success criteria, not rigid step scripts

### Hooks Quick Reference
| Hook Type | Fires | Use For |
|---|---|---|
| Pre-tool-use | Before tool executes | Blocking disallowed calls, enforcing prerequisites, redirect-not-just-block |
| Post-tool-use | After tool executes, before model sees output | Normalizing/cleaning data format |

### Workflow Strategy Selection
| Task Type | Strategy |
|---|---|
| Known, predictable steps | **Chain** (fixed sequence) |
| Unknown/discoverable scope | **Adaptive plan** (revise as you learn) |
| Multi-file/multi-unit review | **Per-file pass, THEN cross-file/integration pass** (never combined into one pass) |
| Multi-concern single request | **Decompose into sub-tasks, run independent ones in parallel** |

### Session Management Decision Tree
```
Is the prior session's context still factually accurate?
├── YES → is there new info since last visit?
│         ├── YES → Resume + explicitly state what changed (targeted resume)
│         └── NO  → Resume as-is
└── NO (stale) → Start fresh, seed with a short summary (not full raw history)

Want to explore a different path without losing the original?
└── Fork (copy shared state, diverge from there)
```

### Common Anti-Patterns = Exam Wrong-Answer Signatures
- Stopping/continuing a loop based on text keywords instead of `stop_reason`
- Sub-agents calling each other directly (no coordinator)
- Spawning a sub-agent with a vague task and no packed context
- Coordinator invoking all sub-agents regardless of query relevance
- Overlapping or gapped task decomposition across sub-agents
- Single-pass finalization with no gap-check
- Enforcing critical rules only via prompt wording, not code/hooks
- Simple "block" with no redirect/escalation path
- Vague hand-off summaries requiring the receiver to re-derive context
- Combined per-file + cross-file review in one pass (attention dilution)
- Fixed-step chain applied to an open-ended/unknown-scope task
- Resuming a stale session instead of starting fresh with a summary

---
*Domain 1 complete — pending: Domain 2 (Prompt Engineering), Domain 3 (Claude Code), Domain 4 (Tool Design/MCP), Domain 5 (Context Management & Reliability).*
