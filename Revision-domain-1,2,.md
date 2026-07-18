# CCA-F QUICK READ — Domains 1-3 (Pre-Mock Review)
*10-15 min skim. Full detail in the 3 domain files if you need to double back.*

---

# DOMAIN 1: Agentic Architecture (27%)

### The Loop
- **`stop_reason`** is the ONLY reliable signal to continue/stop a loop. Values: `tool_use` (run tool, append result, call again), `end_turn` (done), `max_tokens`, `stop_sequence`.
- ❌ Never stop/continue based on keywords in text ("done", "finished") — probabilistic, unreliable.
- Tool result MUST be appended to history before next call — Claude has no memory of tool output otherwise.
- Let Claude pick the next tool dynamically — don't hardcode a fixed step sequence.

### Multi-Agent Design
- **Hub-and-spoke**: all comms through coordinator; sub-agents NEVER call each other directly.
- Sub-agents start with **zero memory** — coordinator must pack topic + prior findings + goal into the spawn prompt.
- Coordinator delegates **selectively** — not every sub-agent for every query.
- **Split without gaps** — non-overlapping task decomposition, full coverage.
- **Check → refill → repeat** — don't finalize on first pass; gap-check then dispatch follow-ups.
- **Task tool must be enabled** for a coordinator to spawn sub-agents at all (no task tool = no delegation, regardless of prompt wording).
- **Fork once, explore many** — do shared/common analysis once, branch for divergent options (avoid redoing base work per branch).
- **Parallelize independent tasks**, sequence dependent ones.
- **Set goals, not fixed steps** — lets sub-agent adapt if a step fails.
- Keep sources attached to facts (traceability).

### Workflow Enforcement
- **Code beats prompt** — critical rules (refund verification, spend limits) enforced in code/tool logic, NOT just prompt wording (prompts are probabilistic, code is guaranteed).
- **Guard the critical step** — tool refuses to run without prerequisite met.
- **One request, many concerns** → decompose into sub-tasks, run independent ones in parallel.
- **Clean hand-off** — escalation payload has full structured context, not vague pointer.

### Task Decomposition
- Known/predictable path → **chain** (fixed sequence). Unknown/discoverable → **adaptive plan** (revise as you learn).
- **Per-file first, THEN cross-file** — never combine local + integration review in one pass (causes missed/contradictory findings).

### Session Management
- **Name it, resume it** — persisted context, pick up where you left off.
- **Fork** — shared baseline, branch to explore alternate paths without redoing common work.
- **Targeted resume** — tell agent exactly what changed, not full re-sweep.
- **Start fresh when stale** — seed with a short summary, not full raw history (= manual context compaction).

⚠️ **TOP TRAP:** Any scenario where "the agent occasionally skips a rule despite clear prompt wording" → answer is always "move enforcement into code," never "reword the prompt more forcefully."

---

# DOMAIN 3: Tool Design & MCP (18%)

### Tool Selection (description quality)
- Claude picks tools based on **description**, not name.
- Good description = what it does + input format + output format + example + when-to-use-vs-similar-tools.
- **Kill overlap** — distinct names/scopes, no "near-twin" tools (e.g. `resize_image` vs `detect_object`, not `tool_A`/`tool_B` both "handles images").
- **Split generic into specific** — one tool per action (`read_file`/`delete_file`/`rename_file`), not one `manage_file`.
- **Watch the system prompt** — absolute words ("always use X tool") can override good tool descriptions. If well-described tools still misfire → check system prompt first, not descriptions.

### Error Handling
- **`isError`** flag (real MCP field name, exact casing) must ALWAYS be set explicitly on failure — never rely on parsing text content.
- 4 error types: **transient** (retry), **validation** (fix input, don't retry), **business** (explain policy), **permission** (escalate, don't retry).
- Include `error_type` + `retryable` + specific message — not generic "operation failed."
- **Empty ≠ broken** — zero search results = `isError: false` with "no results found." Conflating "not found" with "failed" is a classic bug/trap.
- Plain language to end users — never raw error codes.
- Fix locally (retry) first, escalate only when truly stuck.

### Tool Scoping & `tool_choice`
- **~4-5 tools per agent** — more degrades selection reliability even with good descriptions.
- Give agent only role-relevant tools ("stay in your lane").
- Give high-frequency shared tools locally to sub-agents that need them often (e.g. support agent gets its own `reset_password`) rather than always escalating.
- Replace unconstrained tools (open-any-URL browser) with constrained ones (allowlisted docs).
- **`tool_choice` values:** `auto` (Claude decides, may skip tools), `any` (must call SOME tool), `{"type":"tool","name":"X"}` (forces specific tool), `none` (must not call any tool).
- **Forced-first-tool pattern**: force a specific tool (e.g. `verify_identity`) on step 1, then switch back to `auto`.
- `tool_choice` (API param, blocks selection) ≠ hooks (code, intercepts after Claude already decided) — two different enforcement layers, don't conflate.

### MCP Config
- **`.mcp.json`** = project root, COMMITTED, shared with team.
- **User-level config** (`~/.claude.json` / home dir) = personal, NEVER committed.
- Secrets: reference `${ENV_VAR}` in `.mcp.json`, real values in local `.env`, excluded via `.gitignore`.
- All MCP servers/tools load **simultaneously** at startup (not lazily).
- MCP primitives: **Tools** (actions) + **Resources** (readable context/data, avoids repeated paginated discovery calls) + **Prompts** (not covered in course — known gap).
- Don't rebuild existing servers (GitHub, Jira) — build custom only for unique internal needs.

### Built-in Tools
- **Glob** = filename/pattern search. **Grep** = content/text search inside files.
- **Read** = view. **Write** = full replace. **Edit** = small targeted change.
- If Edit target isn't uniquely identifiable (e.g. "update Rahul" when 4 Rahuls exist) → Read first to disambiguate, then Edit/Write.
- Tracing usages: direct `grep` on a function name misses **wrapper/indirect callers** — must trace the full call chain.

⚠️ **TOP TRAP:** "Wrong tool selected despite good descriptions" → check tool COUNT/scoping (too many tools) or SYSTEM PROMPT override, not description quality again.

---

# DOMAIN 2: Claude Code Configuration & Workflows (20%)

### CLAUDE.md Scoping
| Level | Path | Committed? |
|---|---|---|
| User | `~/.claude/CLAUDE.md` | No |
| Project | `CLAUDE.md` @ project root | Yes |
| Directory | `<folder>/CLAUDE.md` | Yes |

- **Confirmed precedence: Directory > Project > User** (most specific wins).
- User-level does NOT exist by default — must be manually created.
- `@import filename.md` — literal syntax to pull in a shared rules file (avoid duplicating rules across files).
- Split monolithic (1000+ line) CLAUDE.md into topic files (`testing.md`, `deployment.md`) — BUT if content is *occasional/task-specific* (not a standing rule), it belongs in a **Skill** instead, not just a split file.
- **`/memory`** — diagnostic command, lists currently loaded CLAUDE.md files. Use this, don't guess, when Claude seems to ignore rules.
- Claude correctly reports "no visibility" when asked about rules that don't exist anywhere loaded — doesn't fabricate. (Reliability-positive behavior.)

### Path-Scoped Rules
- Rules can be glob-scoped (`**/*.sql`, `**/*.md`) to load ONLY when a matching file type is being edited.
- Match by **file type/pattern**, not by folder location — a directory-scoped rule misses matching files sitting elsewhere.
- One glob rule > copy-pasting the same rule into every subdirectory.
- Framing: this is fundamentally **context-budget management** (connects to Domain 5).

### Team vs. Personal Scoping (pattern repeats 3x)
| Mechanism | Team (shared) | Personal |
|---|---|---|
| CLAUDE.md | `./CLAUDE.md` | `~/.claude/CLAUDE.md` |
| Commands | `.claude/commands/` (project) | `.claude/commands/` (home) |
| Skills | `.claude/skills/` (project) | `.claude/skills/` (home) |

Rule: `./.claude/` = shared/committed. `~/.claude/` = personal/never committed. Same pattern every time.

### Skills (`SKILL.md`)
- **Front matter = contract** — read before instructions execute. Specifies: `allowed-tools`, argument hints, context.
- No `allowed-tools` restriction = skill retains MORE tool access than its stated purpose needs (risk).
- No argument hint = silent failure / Claude has to guess required input.
- Noisy/verbose skills (e.g. "scan project") should run in a **context fork** — main chat gets only a summary.
- **CLAUDE.md = always loaded. Skills = loaded only when invoked.** This is the deciding factor for where content belongs.

### Plan Mode vs. Direct Execution
- **Plan**: multi-file, multiple valid approaches, architectural decisions, high rework risk.
- **Execute directly**: single file, obvious path, trivial change.
- **Explorer pattern**: broad/verbose discovery tasks run isolated; main session sees only summary.
- **Analysis paralysis** = anti-pattern too — repeated re-planning with no execution is also a failure mode, not just skipping planning.

### CI/CD
- **`-p` flag** = non-interactive/print mode. Missing this = CI job hangs waiting for input that never comes.
- **`--output-format json`** = structured output for automated parsing (e.g. auto-posting review comments).
- CI-run Claude needs CLAUDE.md context loaded too, or generated code/tests won't follow team conventions.
- **Reviewer isolation**: the session that WROTE code shouldn't be the same session that REVIEWS it (misses own mistakes) — use a fresh session.
- Don't re-review/re-test from scratch every run — pass forward prior findings / existing test files, review only what's new.

⚠️ **TOP TRAP:** "Team ignores my rule" → almost always a SCOPE problem (rule in personal/uncommitted file instead of project-level committed file), diagnosed via `/memory`.

---

# 🔑 THE 5 PRINCIPLES THAT REPEAT ACROSS EVERY DOMAIN
(If you remember nothing else, remember these — they're tested in every domain's clothing)

1. **Code/parameter beats prompt** — hooks, `tool_choice`, tool-level validation, code-enforced rules > wording something more strongly in a prompt.
2. **Narrow scope = reliable** — fewer tools per agent, per-file-then-cross-file review, `allowed-tools`, task decomposition — smaller/focused always beats broad/generic.
3. **Fresh eyes / independent review** — coordinator gap-checking, per-file+cross-file passes, CI reviewer session isolation — the entity that did the work shouldn't be the sole judge of it.
4. **Personal (local/uncommitted) vs. team (shared/committed)** — identical pattern across CLAUDE.md, commands, skills, MCP config.
5. **Don't guess — verify or ask** — `stop_reason` not keywords, `/memory` not guesswork, "interview before building" not assuming requirements, Claude reporting "no visibility" instead of fabricating.

---

# COMMAND / SYNTAX CHEAT-SHEET (exact recall risk)
```
stop_reason values:        tool_use | end_turn | max_tokens | stop_sequence
tool_choice values:        auto | any | {"type":"tool","name":"X"} | none
MCP error field:           isError  (exact casing)
MCP error categories:      transient | validation | business | permission
CLAUDE.md import syntax:   @import filename.md
CLAUDE.md precedence:      directory > project > user
Diagnostic command:        /memory
CI non-interactive flag:   -p
CI structured output:      --output-format json
Verify CC install:         claude --version
Built-in tools:             Glob (name) | Grep (content) | Read | Write | Edit
Tools-per-agent guideline: ~4-5
MCP config (team):         .mcp.json  (project root, committed)
MCP config (personal):     ~/.claude.json  (home dir, not committed)
```

# STILL-OPEN GAPS (verify independently if time permits)
- `/compact` command (mentioned, never covered)
- MCP **prompts** primitive (3rd primitive, never covered — tools ✅ resources ✅ prompts ❌)
- Exit-code conventions for CI/CD
- Exact `Edit` tool parameter schema (`old_string`/`new_string`?)
