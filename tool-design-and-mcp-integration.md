# CCA-F Domain 3: Tool Design & MCP Integration
## Consolidated Study Notes (18% of exam)

---

## 1. Tool Selection — 5 Principles for Correct Tool Picking

**Underlying concept:** Claude selects tools primarily based on their **description** field. Description quality is the single biggest lever for tool-selection accuracy.

| Principle | Mechanism | Situation |
|---|---|---|
| **Description is the signal** | Claude reads the description text to decide which tool fits a request — not the tool's name alone | Two tools named `tool_A`/`tool_B` with vague descriptions ("handles files"/"works with data") → unreliable selection |
| **What belongs in a description** | A complete description states: what it does, input format, output format, an example, when to use it vs. similar tools | `translate_text`: "Translates English→Hindi. Input: English string. Output: Hindi string. Example: 'hello'→'namaste'" |
| **Kill the overlap** | Distinct names + non-overlapping scopes prevent "near-twin" confusion | `resize_image` ("changes dimensions") vs. `detect_object` ("finds objects in an image") — not two tools both vaguely "handling images" |
| **Split the generic into specific** | One tool per distinct action/purpose, not one tool covering many actions | `read_file`/`delete_file`/`rename_file` as 3 tools, not one `manage_file` tool |
| **Watch your system prompt** | Absolute language ("always"/"never use X tool") in the system prompt can override good tool descriptions | System prompt says "always use calculator" → calculator invoked even for non-math questions |

**Situation:** A support agent has `check_status` ("checks something") and `get_info` ("gets information") — both vague and overlapping. Fix: rename/redescribe to precise, distinct scopes (e.g., `get_order_status` vs. `get_customer_profile`).

**Exam trap:** A scenario shows well-designed, distinct tool descriptions still being misused — root cause is usually the **system prompt** overriding selection (absolute language), not the tool schema. Don't default to "fix the description" when the description is already fine.

---

## 2. Error Handling — 6 Principles for Automatic Recovery

**Underlying concept:** Tool outputs must give the agent structured, actionable information about failures — not just a pass/fail signal buried in freeform text.

| Principle | Field/Mechanism | Situation |
|---|---|---|
| **Always raise `isError`** | Boolean field on the MCP tool result, independent of `content` text | `{"content": "payment failed"}` with no `isError` → agent may think it succeeded. Fix: `isError: true` |
| **Know your error type** | Categories: **transient, validation, business, permission** — each needs a different response | Transient → retry; validation → fix input; business → explain policy; permission → escalate/request access |
| **Generic errors strain the agent** | Include `error_type`, `retryable`, and a specific description | `{"isError": true, "error_type": "network", "retryable": true, "content": "timeout"}` → agent knows it can safely retry |
| **Speak plainly to the customer** | Internal codes translated to plain language before reaching the end user | "Error 42" → "You've used your daily limit" |
| **Fix locally, escalate the stuck** | Retry/self-correct first; escalate only when truly unresolvable | Transient error → retry automatically; permission error → escalate (retry won't help) |
| **Empty is not broken** | Zero results ≠ failure. `isError: false` for successful searches that found nothing | `search_orders` with 0 matches → `{"isError": false, "content": "No orders found"}`, NOT `isError: true` |

**Situation:** `search_inventory` for an out-of-stock item — correct response is `isError: false, content: "No matching products found"` (operation succeeded, zero matches), never `isError: true` (that would incorrectly signal the search itself failed).

**Confirmed MCP fact:** `isError` is a real, literal field name in the MCP tool-result spec (top-level, alongside `content`). MCP mandates you flag errors via this field but doesn't mandate a specific categorization scheme — `error_type`/`category`/`retryable` are best-practice conventions, not protocol requirements.

**Exam trap:** A zero-result search flagged as `isError: true` — classic conflation of "not found" with "operation failed." Also: retry logic needs a cap (max retries/backoff) — blind infinite retry on `retryable: true` becomes its own reliability problem (links to Domain 5).

---

## 3. Tool Assignment & Scoping (How Many Tools, Which Ones)

**Underlying concept:** Too many or irrelevant tools degrade selection reliability, even when each individual tool is well-described.

| Principle | Mechanism | Situation |
|---|---|---|
| **Fewer tools, sharper choice** | Aim for **~4-5 tools per agent** | An agent with 12+ tools shows degraded selection accuracy even with good descriptions |
| **Stay in your lane** | Give an agent only tools relevant to its role | A writing agent gets `write_block`/`summarize`, not `web_search`/`image_edit` |
| **Cross-roll only when needed** | Give a sub-agent a normally-coordinator-level tool if it handles that request type frequently | Front-line support agent gets its own `reset_password` tool instead of escalating every reset to the coordinator |
| **Generic with constraint** | Replace unconstrained tools with narrower, bounded versions | `open_browser` (any URL) → `view_approved_docs` (allowlisted sites/PDFs only) |

**Situation:** An agent with 18 tools (3 real + 15 near-duplicate "document" tools) exhibits random/unreliable tool selection — root cause is sheer tool-set size and lack of scoping, not description quality (this was directly demonstrated as an anti-pattern in the hands-on lab).

**Exam trap:** Distinguish tool-count/scoping problems from description-quality problems — a scenario may offer both "improve descriptions" and "reduce tool count" as options; pick based on stated root cause (are descriptions already good but there are just too many tools? → scope down. Are there few tools but vague descriptions? → rewrite descriptions).

---

## 4. `tool_choice` — Three (Four) Modes of Control

**Underlying concept:** `tool_choice` is a real Claude API parameter controlling how strictly Claude must use tools — a different enforcement layer than hooks (Domain 1), operating at tool *selection* time rather than intercepting after a call is proposed.

| Value | Behavior | Use Case |
|---|---|---|
| `auto` | Claude decides whether to call a tool at all, and which one | Default, flexible behavior; risk: may skip tool use even when needed |
| `any` | Claude must call *some* tool, but can choose which | Guarantees a tool call happens (e.g., "ticket purchase only" flows) |
| `tool` (forced specific, by name) | Claude must call one named tool | Mandatory first step (e.g., identity verification) that can never be skipped |
| `none` *(not in transcript, confirmed via exam-guide search)* | Claude must not call any tool | Final summarization turn after tool use is complete |

**Situation — forced-first-tool pattern:** A banking agent's first API call in the loop sets `tool_choice = {"type": "tool", "name": "verify_identity"}`, guaranteeing verification always runs first; subsequent calls in the loop revert to `tool_choice: "auto"`.

**Situation — forced `any`:** A ticket-purchase agent should never just reply with text — `tool_choice: "any"` guarantees a tool call every turn, unlike `auto` which could produce a pure-text response.

**Exam-critical distinction:** `tool_choice` forcing (API-parameter-level, controls what Claude *can choose*) vs. **pre-tool-use hooks** (Domain 1, code-level, intercepts *after* Claude has already decided to call a tool). Both achieve "guaranteed critical step" but via different mechanisms/layers — a scenario may ask which layer is appropriate, or present both as options where only one fits the stated constraint.

---

## 5. MCP Server Configuration

**Underlying concept:** MCP servers extend Claude with additional tools/resources, configured at two distinct scope levels, with strict secret-handling discipline.

| Config Location | Scope | Committed? | Contains |
|---|---|---|---|
| **`.mcp.json`** | Project/repository level | Yes — committed, shared with team | Shared server definitions (GitHub, Jira, internal tools everyone needs) |
| **User-level config** (home directory, e.g. `~/.claude.json`) | Personal/individual | No — local only | Personal/experimental servers not meant for the team |

**Secret handling:** `.mcp.json` references secrets via **environment variable placeholders** (e.g., `${GITHUB_TOKEN}`), never literal values. Actual values live in a local `.env` file, excluded from commits via `.gitignore`. This lets the server *definition* be shared while each developer's actual credentials stay private.

**Startup behavior:** All configured MCP servers and their tools load **simultaneously** at Claude Code startup — not lazily/sequentially. Full toolset is available immediately.

**Description quality applies to MCP tools too — including vs. built-in tools:** A vague MCP tool description (e.g., "search file") can lose out to Claude's own built-in tools even when objectively more powerful, because selection is description-driven regardless of tool source.

**Build vs. reuse:** Use existing/community MCP servers for common integrations (GitHub, Jira, Slack); reserve custom server development for genuinely unique internal needs (e.g., proprietary billing systems) with no existing equivalent.

**MCP Resources (distinct primitive from tools):** Resources expose readable data/context (schemas, summaries, doc hierarchies) an agent can consult directly — avoiding repeated exploratory tool calls (e.g., paginated "list issues page 1, 2, 3..." calls). Tools = invocable *actions*; Resources = readable *context*.

**Situation:** A team needs shared GitHub PR/issue access, with each developer having their own personal access token. `.mcp.json` (committed) defines the GitHub server referencing `${GITHUB_TOKEN}`; each developer's actual token lives in their local, gitignored `.env`.

**Gap flagged for independent study:** MCP's third core primitive, **prompts** (reusable prompt templates a server can expose), was never covered by this course — tools and resources were both covered, prompts were not.

---

## 6. Claude Code Built-In Tools — Grep, Glob, Read/Write/Edit

**Underlying concept:** Disciplined, efficient file exploration and modification — avoid brute-force reading everything; use the right tool for content-search vs. name-search vs. modification-scope.

| Tool | Searches/Acts On | Use When |
|---|---|---|
| **Grep** | File *content* (pattern/keyword inside files) | You know *what* text you're looking for, not which file it's in |
| **Glob** | File *names* (pattern matching) | You know the file name/type pattern, not the content |
| **Read** | Full file content (view) | You need to see a file's complete current content |
| **Write** | Full file replacement | You're creating a new file or intentionally replacing all content |
| **Edit** | Small, targeted change to a specific part | You're making a minor, precisely-anchored change |

**Edit fallback rule:** If the edit target can't be **uniquely identified** (e.g., "update Rahul's mark" when 4 students are named Rahul), don't blindly Edit — Read first to disambiguate (e.g., find "Rahul Sharma, roll #27"), then Edit/Write only the correct, uniquely-identified target.

**Incremental exploration:** Start from the most likely lead (e.g., a Grep result pointing to `login.py`) and follow it, rather than reading every file "just in case" upfront. Same underlying principle as Domain 1's "adapt as you discover," applied to codebase exploration.

**Trace through wrappers:** When searching for all usages of a function, a direct search (e.g., `grep "send_email("`) only finds *direct* calls — indirect callers via wrapper functions (`notify_user()` → `mail_customer()` → `send_email()`) can be missed. Trace the full call chain before concluding you've found every usage.

**Situation:** Task: "update the discount field for the customer record," but Grep shows "discount" appears in 6 different records. Correct approach: Read to find the specific, uniquely-identifiable record (by customer ID), then Edit/Write only that one — never guess or blanket-apply.

**Cost/efficiency note (beyond transcript):** `Write` requires generating the entire file's new content as output — far more token-expensive than a targeted `Edit`, especially for large files. This is the concrete "why" behind preferring Edit when the change is small and the target is unique.

---

# 🎯 EXAM-DAY CHEATSHEET — Domain 3

### Core Meta-Principle
**Description quality drives tool selection. Scope drives reliability. `isError` + categorization drives recoverability.** Three separate levers — don't conflate them when diagnosing a scenario's root cause.

### Root-Cause Diagnosis Table (for "wrong tool was selected" scenarios)
| Symptom | Likely Root Cause | Fix |
|---|---|---|
| Two tools both plausible for a request | Vague/overlapping descriptions | Rewrite descriptions: what/input/output/example/when-to-use |
| Right tool exists but ignored, wrong tool always used instead | System prompt has absolute language ("always use X") | Remove "always"/"never," rely on descriptions |
| Selection degrades as more tools are added, even with good descriptions | Tool-set too large / unscoped | Reduce to ~4-5 tools per agent; split into role-specific agents |
| A critical step (e.g., verification) sometimes skipped | No forcing mechanism | Use `tool_choice: {"type":"tool","name":"X"}` for that step |
| Model never calls a tool when action is required | `tool_choice: auto` risk | Switch to `tool_choice: "any"` |
| Custom/MCP tool ignored in favor of a built-in tool | MCP tool's description is vague relative to built-in | Rewrite MCP tool description to state specific capability/advantage |

### `isError` / Error Response Quick Reference
```
{
  "isError": true,               // ALWAYS set explicitly on failure — never rely on content text alone
  "error_type": "transient" | "validation" | "business" | "permission",
  "retryable": true | false,
  "content": "specific, actionable description"
}
```
- Zero results ≠ error → `isError: false` with a "no results found" message.
- Transient + retryable → retry automatically (with a cap).
- Validation → don't retry blindly; fix input.
- Permission → escalate, don't retry.
- Business → explain the rule in plain language to the user.

### `tool_choice` Quick Reference
| Value | Claude Must... |
|---|---|
| `auto` | Decide freely (may skip tools entirely) |
| `any` | Call *some* tool (not necessarily a specific one) |
| `{"type":"tool","name":"X"}` | Call exactly tool X |
| `none` | Not call any tool |

**Pattern:** Force specific tool for mandatory first step → `auto`/`any` for flexible middle steps → `auto`/`none` for final synthesis.

### MCP Config Quick Reference
| File | Location | Shared? | Purpose |
|---|---|---|---|
| `.mcp.json` | Project root, committed | Yes | Team-wide MCP servers |
| User-level config | Home directory | No | Personal/experimental servers |

**Secrets:** Never hardcode in `.mcp.json`. Use `${ENV_VAR}` placeholders; real values in local `.env`, excluded via `.gitignore`.

**MCP primitives:** Tools (actions) · Resources (readable context/data) · Prompts (reusable templates — least covered, verify independently).

### Built-In Tool Selection Quick Reference
```
Know the file NAME/pattern, not content?     → Glob
Know the CONTENT you're searching for?        → Grep
Need to VIEW a file fully?                    → Read
Creating new / fully replacing a file?        → Write
Small, precisely-anchored change?             → Edit
Edit target not uniquely identifiable?        → Read first, then Edit/Write
Investigating all usages of a function?       → Trace wrapper/indirect calls, don't stop at first direct match
```

### Common Anti-Patterns = Exam Wrong-Answer Signatures
- Vague, overlapping tool descriptions ("analyze_content" vs. "analyze_document")
- System prompt with "always use tool X" overriding sound tool descriptions
- One agent with 15-20 tools, no scoping
- Generic `manage_X` tool covering many distinct actions
- Unconstrained tools (e.g., open-any-URL browser) instead of allowlisted/constrained versions
- Missing `isError` flag; relying on parsing free-text content to detect failure
- Identical generic error message ("operation failed") for every failure type
- Treating a zero-result search as an error
- Internal error codes shown directly to end users
- Escalating every error immediately instead of retry-then-escalate
- No `tool_choice` forcing on a step that must never be skipped
- Hardcoded secrets/API keys directly in `.mcp.json`
- Blind `Edit` on a non-unique target instead of Read-then-disambiguate
- Stopping a "find all usages" search at the first direct call, missing wrapper functions

---
*Domain 3 complete. Pending: Domain 2 (Claude Code Configuration), Domain 4 (Prompt Engineering), Domain 5 (Context Management & Reliability).*

## Cross-Domain Meta-Principles (running, updated through Domain 3)
1. **Code/parameter beats prompt** — enforced across 3 layers now seen: hooks (Domain 1, code-level), `tool_choice` forcing (Domain 3, API-parameter-level), tool-level validation (Domain 1, business logic inside the tool implementation).
2. **Narrow scope = more reliable** — seen at 3 granularities: sub-agent task splitting (Domain 1), per-file vs. cross-file review passes (Domain 1), tool-set scoping per agent (Domain 3).
3. **Adapt as you discover / incremental exploration** — seen in: task decomposition strategy (Domain 1), dynamic planning hands-on (Domain 1), Grep-then-Read file exploration (Domain 3).
4. **Good vs. anti-pattern paired teaching** — nearly every concept across Domains 1 and 3 was taught via an explicit "good implementation vs. labeled anti-pattern" contrast — exam distractors are very likely modeled directly on these demonstrated anti-patterns.

## Gaps Tracker (as of end of Domain 3)
- MCP **prompts** primitive — never covered by the course (tools ✅, resources ✅, prompts ❌).
- Exact `Edit` tool parameter names (e.g., `old_string`/`new_string`) — conceptually covered, exact schema not confirmed; verify independently.
- CLAUDE.md configuration hierarchy — belongs to Domain 2 (Claude Code), not yet started.
- Retry-cap/backoff mechanics for `retryable: true` errors — mentioned as a natural extension, not covered in depth; likely connects to Domain 5 (Context Management & Reliability).
