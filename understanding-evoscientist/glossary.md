# Glossary

One entry per concept introduced in this book: the term, a one-line gloss (as given on first
occurrence), and the chapter that owns its full definition. Terms are in alphabetical order.
Where a term is a borrowed-framework concept (LangChain/LangGraph/deepagents), the gloss
describes how this book uses it; where it is EvoScientist-specific, the gloss is the project's
own vocabulary. As everywhere in this book: **when the glossary and the code disagree, the code
wins** — treat this page as an index into the chapters, not a substitute for them.

---

**Async sub-agent** — a sub-agent run as a separate remote graph in the `langgraph dev`
subprocess, so delegation is non-blocking. *(introduced in Chapter 6)*

**AsyncSqliteSaver** — the LangChain-provided SQLite checkpointer base class (tables
`checkpoints` + `writes`) that EvoScientist's `PruningCheckpointer` subclasses. *(introduced in
Chapter 13)*

**Auxiliary model** — a cheaper/faster model used for background work: memory workers, the tool
selector, the scheduler sub-agent. *(introduced in Chapter 5)*

**AutoSkills** — the process that mines skill proposals from clusters in the observation graph,
for human approval. *(named in Chapter 1; introduced in Chapter 12)*

**Backend (`BackendProtocol`)** — the uniform file+shell interface (read/write/edit/ls/grep/glob/
execute) that the agent's tools run against. *(introduced in Chapter 4)*

**Background process** — a detached OS process launched via `run_in_background`; in-memory only
and lost on restart by design; distinct from an async task and a cron schedule. *(introduced in
Chapter 14)*

**ccproxy** — an external local proxy that lets subscription OAuth tokens stand in for an API key.
*(introduced in Chapter 8)*

**Channel** — one chat-platform integration (Telegram, Slack, …) behind a common `Channel` base
class. *(introduced in Chapter 15)*

**Chat model** — the LLM wrapped behind a uniform interface that takes messages and returns a
message. *(introduced in Chapter 3)*

**Checkpointer** — the component that snapshots graph state after each super-step so a run can
resume. *(introduced in Chapter 3; EvoScientist's own checkpointer is Chapter 13's
`PruningCheckpointer`)*

**Code interpreter** — the sandboxed QuickJS (JavaScript) REPL exposed to the model as a tool.
*(introduced in Chapter 9)*

**Conditional edge** — a router that picks the next node from the current state (here: are there
tool calls?). *(introduced in Chapter 3)*

**Context editing** — a middleware that clears old tool-call/result pairs once history passes
~50% of the context window. *(introduced in Chapter 7)*

**`create_agent`** — LangChain's factory that compiles a ReAct agent into a runnable graph.
*(introduced in Chapter 3)*

**`create_deep_agent`** — deepagents' factory that adds a filesystem, sub-agents, skills, and
summarization on top of `create_agent`. *(introduced in Chapter 4)*

**CompositeBackend** — a backend that routes calls by path prefix to other backends (`/skills/`,
`/memories/`, default). *(introduced in Chapter 4)*

**Dangerous mode** — a flag that drops workspace confinement for real-filesystem access (the
command blocklist still applies). *(introduced in Chapter 9)*

**Deep agent** — an agent with a virtual filesystem, sub-agent delegation, and skills, built by
deepagents. *(introduced in Chapter 4)*

**deepagents** — the "batteries-included" agent framework wrapping `create_agent`; EvoScientist
configures it. *(introduced in Chapter 4)*

**Delta channel** — LangGraph's message storage strategy: periodic full snapshots plus
incremental writes, replayed on read. *(introduced in Chapter 13; named in Chapter 4)*

**Deterministic dedupe ID** — an observation's ID, a content hash (`O-` + first 16 hex chars of a
SHA-256 over memory type, scope, and normalized text) so re-recording the same insight is a
no-op. *(introduced in Chapter 11)*

**Error normalization** — a middleware that wraps heterogeneous provider exceptions into one
serializable error type. *(introduced in Chapter 7)*

**EvoMemory** — EvoScientist's self-evolving memory: Markdown files distilled from each turn and
linked across sessions into a knowledge graph; not a vector database. *(named in Chapter 1;
introduced in Chapter 11)*

**Gateway (`GraphGateway`)** — the seam giving every surface (CLI, WebUI, chat channels) one
thread/stream API over local in-process or remote `langgraph dev` graph execution. *(introduced
in Chapter 15)*

**HITL (human-in-the-loop)** — pausing before a risky tool (e.g. `execute`) for human approval;
implemented as middleware rather than a graph-level kwarg. Distinct from human-on-the-loop.
*(named in Chapter 1; mechanism introduced in Chapter 7)*

**Human-on-the-loop** — the product stance: the human reviews direction at checkpoints, not
every action. *(introduced in Chapter 1)*

**`interrupt()`** — LangGraph's pause-for-human primitive; a run is resumed later with
`Command(resume=…)`. *(introduced in Chapter 3)*

**langgraph dev** — the local LangGraph API server that EvoScientist runs as a child process,
hosting async sub-agents, EvoMemory's background workers, and cron-fired scheduled tasks.
*(introduced in Chapter 14)*

**LangChain** — the LLM-application framework EvoScientist builds on; provides `create_agent`,
the `@tool` decorator, and the message types. *(introduced in Chapter 3)*

**LangGraph** — the state-machine library underneath LangChain; models computation as a graph of
nodes over shared state. *(introduced in Chapter 3)*

**Lazy factory** — building the agent only on first use, so fast CLI commands (like `--help`)
don't pay the assembly cost. *(introduced in Chapter 5)*

**MCP (Model Context Protocol)** — an open protocol for plugging external tool servers into the
agent. *(introduced in Chapter 10)*

**Memory preflight** — the agent's prompt-mandated "check memory before acting" routine: consult
the inlined index, then `search_observations`, then `read_memory`. *(introduced in Chapter 11)*

**Memory type** — an observation's kind: `semantic` (facts), `procedural` (how-to), or `episodic`
(events) — borrowed from cognitive science's vocabulary for human long-term memory. *(introduced
in Chapter 11)*

**Memory worker** — a background LLM agent, on the auxiliary model, that distills a finished turn
into observations. *(introduced in Chapter 11)*

**Middleware (`AgentMiddleware`)** — a class that hooks the agent's lifecycle to intercept or
modify model and tool calls. *(introduced in Chapter 4)*

**Middleware hook** — one interception point on the middleware lifecycle: `wrap_model_call`,
`wrap_tool_call`, `modify_request`, `after_agent`, and their async counterparts. *(introduced in
Chapter 4)*

**MiddlewareEventSink** — a dependency-injection seam (a `Protocol`) that lets frontends observe
middleware activity (e.g. streaming progress) without middleware depending on any one frontend.
*(introduced in Chapter 7)*

**Model fallback** — a middleware that retries a failed model call down a configured chain of
models. *(introduced in Chapter 7)*

**Node** — one step in a `StateGraph` (e.g. the `"model"` node, the `"tools"` node). *(introduced
in Chapter 3)*

**Observation** — one atomic memory note: a Markdown file (`O-<hash>.md`) with YAML frontmatter
(id, summary, memory type, scope, source, related observations) and a short body. *(introduced
in Chapter 11)*

**Observation linker** — a background LLM agent that adds relation edges between observations.
*(introduced in Chapter 11)*

**Observation relation** — a typed edge between observations, stored in frontmatter: `complements`
/ `contradicts` / `supersedes` (the last is directional; its reverse edge is suppressed).
*(introduced in Chapter 11)*

**onion / middleware stack** — the ordered list of middleware wrapping each model or tool call,
outermost first; order determines behavior. *(introduced in Chapter 4; the full stack is Chapter
7's subject)*

**Provider routing** — switching LLM vendors by pointing the same SDK at a different base URL and
API key. *(introduced in Chapter 8)*

**PruningCheckpointer** — EvoScientist's checkpointer, subclassing `AsyncSqliteSaver`, that prunes
old rows so `sessions.db` stays bounded without breaking resume. *(introduced in Chapter 13)*

**PTC (pass-through call)** — read-only, batchable tools exposed as JS-callable functions inside
the QuickJS code interpreter, for `Promise.all`-style fan-out. *(introduced in Chapter 9)*

**ReAct loop** — the reason → act → observe cycle: model → (if tool calls) tools → model → … →
stop. *(introduced in Chapter 3)*

**Reasoning / thinking** — a model's hidden chain-of-thought, controlled per-vendor via
effort/budget knobs. *(introduced in Chapter 8)*

**Recursion limit** — the maximum number of super-steps before LangGraph aborts a run, guarding
against infinite tool loops. *(introduced in Chapter 3)*

**Sandbox / workspace** — the confined virtual `/` the agent operates in by default; auto-
corrects LLM absolute-path mistakes and rejects escapes. *(introduced in Chapter 9)*

**Skill / `SKILL.md`** — a self-contained capability package: a directory with a `SKILL.md` (YAML
frontmatter with `name` + `description`, plus Markdown instructions) and optional supporting
files; loads into context only when triggered ("progressive disclosure"). *(named earlier;
introduced in Chapter 12)*

**Slash command** — a user control message (e.g. `/model`, `/schedule`) parsed separately from
ordinary prompt text. *(introduced in Chapter 10)*

**State / reducer** — the shared data threaded through a `StateGraph`; a reducer says how each
node's update merges into that state. *(introduced in Chapter 3)*

**StateGraph** — a LangGraph graph: a set of named nodes (functions) that read and write one
typed shared state object. *(introduced in Chapter 3)*

**Streaming event** — one normalized dict (with a `"type"` discriminator: `text` / `tool_call` /
`interrupt` / …) emitted from `astream_events(v3)` via `StreamEventEmitter`, forming a closed
vocabulary every surface understands. *(introduced in Chapter 15)*

**Sub-agent** — a specialized agent (planner / research / code / debug / data-analysis / writing,
plus the scheduler) that the main agent delegates to via the task tool. *(concept introduced in
Chapter 4; roster introduced in Chapter 6)*

**Super-step** — one round of the graph advancing all its active nodes; the unit a checkpoint is
saved after. *(introduced in Chapter 3)*

**System prompt** — the standing instructions prepended to every model call; EvoScientist's is
built as a four-source stack and is described in the book as its "constitution." *(introduced in
Chapter 5)*

**Task tool** — the tool the main agent calls to delegate to a sub-agent; runs a nested,
stateless sub-graph in one parent tool call. *(mechanism introduced in Chapter 4; usage detailed
in Chapter 6)*

**Thread / `thread_id`** — the id grouping one conversation's chain of checkpoints. *(introduced
in Chapter 3)*

**Tier-aware mounts** — skills merged from workspace > global > builtin tiers into one `/skills/`
namespace, with the higher tier shadowing the lower by name. *(introduced in Chapter 12)*

**Tool call** — a structured request the model emits, asking the runtime to run a named function.
*(introduced in Chapter 3)*

**Tool selector (adaptive tools)** — a middleware that, above roughly 26 tools, uses an extra LLM
call to pick a relevant subset for the turn. *(introduced in Chapter 7)*

**ToolNode** — LangGraph's built-in node that executes the tool calls found in the last model
message. *(introduced in Chapter 3)*

**uv** — Astral's Rust-based Python package and project manager, used for dependency resolution,
virtual environments, builds, and running. *(introduced in Chapter 16)*

**vibe research** — the project's brand term for loosely-directed, agent-led exploratory
research — the opposite of a rigid checklist. *(introduced in Chapter 1)*

**Virtual filesystem** — the `/`-rooted paths the agent sees, mapped onto real storage by a
backend. *(introduced in Chapter 4)*

---

## Supporting terms

These are secondary terms used consistently across chapters, each owned by the same chapter as
the primary concept it supports.

**`after_agent` / `wrap_model_call` / `wrap_tool_call` / `modify_request`** — the specific
middleware hooks named above; see **Middleware hook**. *(Chapter 4)*

**`astream_events(version="v3")`** — LangGraph's asynchronous streaming interface; the raw
protocol Chapter 15's `_V3EventProcessor` translates into the closed streaming-event vocabulary.
*(Chapter 3, vocabulary payoff in Chapter 15)*

**BLOCKED_COMMANDS** — the hand-rolled command blocklist enforced regardless of dangerous mode.
*(Chapter 9)*

**`Command(resume=…)`** — the call that resumes a graph paused by `interrupt()`. *(Chapter 3)*

**Connected component** — a maximal set of mutually-reachable observations in the memory graph;
AutoSkills candidates are exactly the connected components of that graph. *(Chapter 12)*

**Cron (scheduled task)** — a recurring run; in EvoScientist a scheduled task IS a LangGraph cron
targeting the scheduler sub-agent's graph — there is no bespoke scheduler. *(Chapter 14)*

**Defense-in-depth** — the security posture behind dangerous mode and the sandbox: multiple
independent safeguards, so no single flag removes every protection. *(Chapter 9)*

**Handler** — the callback a `wrap_*` middleware hook may call, short-circuit, or rewrite around.
*(Chapter 4)*

**HarnessProfile** — deepagents' mechanism for per-model tuning of prompt, tools, and middleware.
*(named in Chapter 4)*

**MemoryScheduler** — the component that batches newly recorded observation IDs and launches the
observation linker only once no memory worker is still active for that batch. *(Chapter 11)*

**Progressive disclosure** — the skill-loading discipline: only a skill's `name` and `description`
are always in context; the full body loads only when the skill triggers. *(Chapter 12)*

**QuickJS** — the JavaScript engine embedded as EvoScientist's sandboxed code interpreter. *(Chapter 9)*

**RunRequest / GraphTarget** — the gateway's normalized description of a request to run, local or
remote, that every surface constructs identically. *(Chapter 15)*

**TF-IDF search** — the from-scratch, field-weighted keyword ranking (`memory/search.py`) that
stands in for embeddings-based semantic search over observations. *(Chapter 11)*
