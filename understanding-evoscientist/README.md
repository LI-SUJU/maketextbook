# Understanding EvoScientist: How a Self-Evolving AI Scientist Is Built

EvoScientist is a CLI-first, multi-agent "AI Scientist" that automates the research lifecycle —
survey, plan, run experiments, debug, analyze, write — while a human stays on the loop rather
than approving every step. This book is a from-source explanation of how it is built. Its central
claim, and the lens for reading every chapter, is that EvoScientist is almost entirely
**configuration**: a chat model wrapped in a borrowed agent loop (LangChain + `deepagents` +
LangGraph), customized with a middleware onion, a team of sub-agents, and a system prompt — plus
two things a plain configured agent still lacks and this project builds for real: a memory that
grows across sessions (**EvoMemory**) and a skill set that mines new capability out of that memory
(**AutoSkills**), both kept under human review. The book teaches the borrowed foundations from
zero (no prior LangChain/LangGraph knowledge assumed), then descends layer by layer through
EvoScientist's own code, and closes by tracing one real research request through every mechanism
it taught. It is written for intermediate-to-strong Python engineers who want to *understand and
extend* a real, production-grade LLM agent framework — whether to contribute to EvoScientist or to
build something like it.

See [`preface.md`](preface.md) for the full narrative arc, what you'll be able to do by the end,
and how the book is structured. See [`glossary.md`](glossary.md) for every term the book
introduces, alphabetically, with the chapter that owns its definition.

## How to read this book

The chapters are ordered so that every concept is taught before it is used — you can read
cover to cover with no unresolved forward references. But depending on why you're here, one of
these four routes may get you what you need faster.

**Route 1 — The ten-minute overview.**
[Ch 1](01-why-evoscientist-exists.md) → [Ch 2](02-the-whole-system.md) → [Ch 17](17-capstone-one-request.md)
Read the motivation, the whole-system map, and the capstone trace. You'll finish knowing *what*
EvoScientist is and *how it hangs together*, without descending into any single implementation.
Good for evaluating the project or briefing a colleague.

**Route 2 — "I want to build an agent framework."**
[Ch 3](03-how-an-agent-loop-works.md) → [Ch 4](04-deepagents-and-middleware.md) → [Ch 5](05-assembling-the-agent.md) → [Ch 7](07-the-middleware-stack.md) → [Ch 9](09-backends-and-sandbox.md)
The load-bearing spine for engineers: how an agent loop actually works, what `deepagents` adds and
the middleware idiom, how EvoScientist assembles its agent, the middleware stack where the real
customization lives, and backends plus safe code execution. These five chapters teach you to build
systems *like* EvoScientist, not just to read this one.

**Route 3 — "Show me the self-evolving magic."**
[Ch 1](01-why-evoscientist-exists.md) → [Ch 11](11-evomemory.md) → [Ch 12](12-autoskills.md)
Why self-evolution matters, then the two mechanisms that make it real: EvoMemory (a knowledge
graph made of Markdown files that grows every turn) and AutoSkills (which mines that graph into
new reusable skills under human approval).

**Route 4 — Cover to cover.**
Start at [Chapter 1](01-why-evoscientist-exists.md) and read straight through. This is the
slowest and most complete route, and the one that ends with you able to add a provider, write a
middleware, define a sub-agent, or debug a stuck checkpoint — and understand *why* each piece is
shaped the way it is.

## Table of contents

### Part I — Orientation
- [Chapter 1 — Why EvoScientist Exists](01-why-evoscientist-exists.md)
  Why a single-shot chatbot can't sustain research, and what "self-evolving" and "human-on-the-loop" concretely mean.
- [Chapter 2 — The Whole System on One Page](02-the-whole-system.md)
  The three-layer stack and the master diagram that every later chapter zooms into.

### Part II — Borrowed Foundations
- [Chapter 3 — How an Agent Loop Actually Works](03-how-an-agent-loop-works.md)
  LLM tool-calling, the ReAct loop, and LangGraph's `StateGraph`/checkpointer/thread — taught from zero.
- [Chapter 4 — deepagents and the Middleware Onion](04-deepagents-and-middleware.md)
  What "deep agents" add for free, and the middleware idiom that is this book's central device.

### Part III — The Core Engine
- [Chapter 5 — Assembling the Agent and Its Constitution](05-assembling-the-agent.md)
  The lazy factory, the kwargs dict that *is* "configure, don't build," and the system prompt as a four-source stack.
- [Chapter 6 — The Team: Sub-Agents and Delegation](06-the-subagent-team.md)
  The six research sub-agents plus the scheduler, and why the same YAML can run two different ways.
- [Chapter 7 — The Middleware Stack: Where the Cleverness Lives](07-the-middleware-stack.md)
  The ~13-layer onion — adaptive tools, context editing, model fallback, HITL-as-middleware — walked whole.

### Part IV — Talking to Models, Running Code
- [Chapter 8 — One Config, Many Model Providers](08-many-providers.md)
  How one flag switches LLM vendors, and the patch farm that keeps two SDKs honest.
- [Chapter 9 — Backends, the Sandbox, and Running Code Safely](09-backends-and-sandbox.md)
  Where file and shell calls actually go, what dangerous mode removes (and never removes), and the QuickJS code interpreter.
- [Chapter 10 — Tools, MCP, and Configuration](10-tools-mcp-config.md)
  Hand-assembled tools, MCP as pluggable external servers, and the config precedence ladder.

### Part V — The Self-Evolving Core
- [Chapter 11 — EvoMemory: A Knowledge Graph Made of Markdown Files](11-evomemory.md)
  Why files, not a vector database; observations, dedupe, relations, and the background distillation loop.
- [Chapter 12 — AutoSkills: Mining New Skills from Memory](12-autoskills.md)
  Skills as `SKILL.md` packages, tier-aware mounts, and skill candidates as connected components of the memory graph.

### Part VI — State, Scale, Surfaces
- [Chapter 13 — Making Conversations Survive: Persistence](13-persistence.md)
  The checkpointer, why naive persistence explodes, and the delta-channel pruning trap.
- [Chapter 14 — Scheduling, Background Work, and Deployment](14-scheduling-and-deployment.md)
  The one `langgraph dev` subprocess behind every unattended actor, and the task/schedule/process distinction.
- [Chapter 15 — One Engine, Many Surfaces](15-one-engine-many-surfaces.md)
  How the CLI, WebUI, and ten chat channels are all thin renderers over one gateway and one event vocabulary.

### Part VII — Building & Contributing
- [Chapter 16 — Build, Test, and Ship](16-build-test-ship.md)
  `uv`, the crown-jewel technique for testing an LLM agent without an LLM, and pins as an incident log.

### Part VIII — Capstone
- [Chapter 17 — One Request, End to End](17-capstone-one-request.md)
  A single real research request traced through every mechanism the book taught.

## Conventions

- **Code citations** are given as `path:line` relative to the EvoScientist repository root, e.g.
  `EvoScientist/memory/observations/store.py:90`.
- **When the book and the code disagree, the code wins.** This book teaches you to read the
  repository, not to replace it; treat every claim as an invitation to go look.
- This book was written against **EvoScientist v0.2.4**.
