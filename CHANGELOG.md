# Changelog

---

<details open>
<summary><strong>v0.4.0</strong> - Three-tier execution approval + Bilingual UI + TUI container layout + Knowledge base BM25</summary>

- **Execution approval three-tier mode** - `safety.permission_mode` supports `ask` (default, y/N confirmation per command), `auto_review` (read-only command whitelist bypass, high-risk commands still require confirmation), and `full_access` (all commands run without confirmation). TUI features synchronous blocking execution approval dialogs with countdowns. Models perform self-risk evaluation before execution. Execution gates and child process spawning are comprehensively hardened. Switchable via `vulnclaw config set safety.permission_mode <mode>`, the REPL `/mode` command, or the `VULNCLAW_SAFETY_PERMISSION_MODE` environment variable.
- **Fix classic REPL `/mode` command** - The `mode` handler was previously registered but missing from the command dispatch table, resulting in `Unknown skill: /mode`. Now registered with complete help documentation.
- **Fix solve tool idle spin** - When the model stops emitting tool calls, it no longer enters an infinite loop, displaying the model's final response and converging cleanly.
- **ScanMalware built-in remote MCP** - Pre-configured ScanMalware cloud scanning MCP server, enabled with one click in `vulnclaw mcp`.
- **`vulnclaw doctor` probes real tool-calling capability** - Doctor now sends real requests to verify whether the configured LLM endpoint supports tool calling rather than only validating configuration syntax.
- **Fix Windows clipboard disk write** - When pasting API keys in the TUI configuration panel, the Windows branch no longer writes clipboard contents to a temporary file. It now reads child process stdout as base64 and decodes in the parent process, leaving zero disk artifacts.
- **TUI draggable container layout** - Capabilities, status, and findings views are decoupled into independent panes. The container layout supports drag-and-drop reordering, findings can be expanded to view underlying evidence, and sub-agent transcripts stream into the main panel in real time.
- **Knowledge base BM25 retrieval and reranking** - Added bigram tokenization, BM25 ranking, and cross-encoder rerankers, significantly improving query hit rates.
- **HTTPS VPS deployment compose profile** - Web UI supports one-click HTTPS Docker Compose deployment on VPS hosts.
- **First-run setup wizard** - Interactive CLI guide for initial language selection and baseline configuration.
- **OpenRouter provider preset** - Automatically binds the corresponding provider when saving an API key, preventing keys from being stored in the wrong section.
- **Bilingual UI (released with v0.3.9)** - Default language set to English, supporting bilingual interfaces. Tool execution lines, status banners, solve report headings, knowledge base status, context truncation hints, LLM retry/recovery alerts, and reasoning state blocks output in the selected language.
- **Additional fixes** - Fixed auto-review classifier bypass; fixed MCP streamable-http tool count being 0 (pinned `mcp>=1.0,<2.0`); adjusted streaming event loop thresholds on slow CI; addressed missing translation keys, HackerOne scope truncation, credential sanitization, and MCP server name period injection vulnerabilities.

</details>

---

<details open>
<summary><strong>v0.3.9</strong> - Bilingual UI</summary>

- **English / Chinese bilingual interface** - Default language set to English (no longer falls back to Chinese when environment signals are missing). CLI/REPL tool invocation lines, status banners, solve report titles, KB status, context truncation notices, and LLM recovery prompts follow the active language. Switchable via `/language` in REPL, `VULNCLAW_LANG=zh|en`, or `session.language` in config.
- **Fix `/language` command output** - Removed extraneous ASCII letter `f` before confirmation text.
- **Agent English keyword support** - Finding parser, phase detection, CTF determination, input analysis, authentication wall, skill dispatch, and MCP routing tables supplemented with English equivalent signal words. Sub-agent behavior under English prompts matches Chinese mode.
- **Knowledge base status localization** - KB initialization, degradation, and disabled details follow the active language.

</details>

---

<details open>
<summary><strong>v0.3.8</strong> - Sub-agent fan-out + Cold/hot memory + Context budget</summary>

- **Model-driven parallel sub-agent fan-out** - The default solve engine introduces the `spawn_subagents` tool, allowing the primary model to dispatch multiple independent, self-contained attack directions concurrently within a single round. Sub-loops inherit target constraints and existing evidence, disallow recursive fan-out, and operate under strict concurrency and lifecycle budgets. Merged evidence reassigns sequence IDs (`eNNN`) while synchronizing claims, pins, progress signals, and tool-call references.
- **TUI sub-agent live monitoring panel** - Textual TUI receives `spawn/start/progress/finish/batch_done` events via a private JSONL protocol with session tokens, displaying role, status, step count, target, and latest progress for each sub-agent in real time.
- **Cold/hot memory separation** - When session history exceeds 48 messages or 32K tokens, older messages are archived into cold memory JSONL shards (rotated at 64MB, up to 8 shards). The hot context retains only recent complete tool exchanges. A `memory_search` tool retrieves context snippets from cold memory by keyword.
- **Unified context budget and structured compaction** - `context_budget.py` provides a single entry point (`prepare_context()`) across all LLM invocation paths. Compaction preserves recent tool exchange groups and replaces older history with deterministic `[context digest v1]` summaries containing target, scope, verified claims, pinned facts, and evidence references.
- **Hardened sub-agent fan-out security and child process lifecycle** - Numerical bounds added to `SubagentConfig` (capping `max_depth` at 2). Out-of-bounds environment variables are rejected with warnings. TUI log rendering escapes untrusted content to prevent markup injection. Process termination follows a three-stage `terminate -> wait -> kill` sequence with unified signal cleanup.
- **Refactored tool loop context management** - `call_llm_auto` maintains a stable prefix (system prompt + bounded history + task instructions) with a dynamic tool-loop tail, compressing only when the tail exceeds high-water thresholds.
- **Skill reference documentation architecture** - The skill resolver injects optional reference indexes (skill name, description, reference file list, and routing rationale) rather than forcing entire skill bodies into prompts. `load_skill_reference` allows the model to consult documentation on demand.
- **De-imperativized correction layer** - System prompts and the correction layer produce diagnostic notes describing evidence state rather than imperatively commanding specific tools or payloads.
- **Active context evidence working set** - Raw outputs are preserved in `AgentState.evidence` while active context receives bounded high-signal previews. Duplicate raw outputs are injected as `same_as=eXXX` references to minimize context bloat.
- **Fix PHP5 deserialization differential false positives** - `http_probe_batch` records response headers and disables TLS verification by default; `runtime_diff_probe` infers runtime from `X-Powered-By: PHP/5.x` and flags signed-length candidates for mandatory remote verification.
- **Enhanced PHP POP chain memory** - Fixed prompt hints highlighting magic method entrypoints to sinks when `unserialize` appears alongside dangerous functions.
- **Enhanced `fetch` HTTPS compatibility** - Automatic retry with `verify_tls=false` when certificate verification fails.
- **`runtime_diff_probe` tool** - On-demand generation and verification of parser-accepted/filter-missed candidates across regex and PHP serialization boundaries.
- **Model-led solve engine** - Transitioned to a model-driven autonomous flow where the LLM determines next steps, tool calls, `FINAL:` completion, `ASK_USER:` clarification, or `NO_PATH:` termination.
- **Built-in tools** - Added `shell_command`, `http_probe_batch`, automatic source code reconstruction, and automated solve retrospective reports.

</details>

---

<details>
<summary><strong>v0.4.1</strong> (Internal feature release line) - Parallel exploration + Memory engine + Reconnaissance toolchain + MCP streamable-http</summary>

- **Multi-direction parallel exploration** - Solve engine supports concurrent exploration paths with isolated evidence buffers.
- **Agent memory engine** - Shared research state with cross-direction tool invocation logs and deduplication.
- **Reconnaissance toolchain** - Added `js_recon` (JS parsing and API extraction), `unauth_test` (unauthorized endpoint discovery), `dir_enum` (concurrency-tuned directory fuzzing), `space_search` (FOFA, Hunter, Quake, Shodan, ZoomEye, 0.zone), and `subdomain_enum`.
- **MCP streamable-http support** - Added HTTP transport support for MCP servers like Chrome DevTools MCP with lazy connection handling.

</details>

---

<details>
<summary><strong>v0.4.0</strong> (Initial architecture refactor) - Goal-driven solver engine</summary>

- **Goal-driven solver engine** - Replaced fixed-round workflows with a stateful plan-action-observe loop terminating on goal achievement, path exhaustion, or safety budget.
- **Evidence-level anti-hallucination gates** - Real tool outputs serve as the sole source of truth; claimed flags must appear verbatim in execution output.
- **Structured reasoning and adaptive reflexion** - Structured confidence ratings for verified facts, progressive L0-L4 payload escalation, and persistent failure memory.
- **Vulnerability detection plugin system** - Low-coupling plugin runtime with built-in read-only web plugins (security headers, JWT, JS endpoints).

</details>
