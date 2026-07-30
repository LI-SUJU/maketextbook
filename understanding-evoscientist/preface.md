# Preface

This is a book about a codebase, and it exists because the codebase is unusually good at
teaching a lesson most engineers building on top of large language models eventually need:
**the interesting part of an AI agent is almost never the agent loop.**

EvoScientist is a CLI-first, multi-agent "AI Scientist" — a system you point at a research
question and that then surveys the literature, plans experiments, writes and debugs code,
analyzes results, and writes up findings, largely unattended, while a human reviews its
direction rather than approving every keystroke. It would be easy to assume a system like that
must implement some large, novel piece of agent machinery — a bespoke planner, a hand-rolled
reasoning loop, a custom orchestration engine. It implements almost none of that. Underneath,
EvoScientist is a chat model in a `while` loop, borrowed nearly whole from two open-source
libraries (LangChain and its `deepagents` extension), wrapped in roughly a dozen middleware
classes and a system prompt. That is the book's thesis in one line: **configure, don't build.**
The engineering that makes EvoScientist work is not a new kind of loop; it is disciplined,
legible configuration of a general one — plus two things a plain configured agent still lacks,
which the project builds for real: a memory that grows across sessions, and a skill set that
grows with it, both kept under human review.

This book teaches you to read that configuration the way its authors built it, layer by layer,
so that by the end you are not just able to operate EvoScientist — you are able to *extend* it,
and to recognize the same pattern the next time you meet a production LLM agent framework, because
by then you will have built the mental model once, properly, rather than picked it up in fragments
from documentation.

## What you will be able to do

By the last page, you will be able to:

- **Add a new model provider** — because you will have read the registry, the routing tables, and
  the two-SDK bet that make one flag switch vendors (Chapter 8).
- **Write a middleware** — because you will have walked real middleware line by line and understand
  why *order* in the onion is itself a design decision, not an implementation detail (Chapters 4
  and 7).
- **Define a new sub-agent** — because you will have read the YAML loader, the stateless task-tool
  dispatch, and the second mechanism (async, remote) that the very same YAML can resolve to
  (Chapter 6).
- **Debug a stuck checkpoint** — because you will understand what a LangGraph thread and
  checkpointer actually are (Chapter 3), and the specific pruning hazard EvoScientist's own
  checkpointer was built to avoid (Chapter 13).

Along the way you will also understand *why* EvoScientist keeps memory as Markdown files instead
of a vector database (Chapter 11), how it mines that memory into new reusable skills without a
human writing them by hand (Chapter 12), and how one compiled agent ends up answering you
identically whether you're in a terminal, a browser, or a Telegram chat (Chapter 15).

## What the book assumes

You should be a comfortable Python engineer — happy with `async`/`await`, classes, decorators, and
subprocesses — with a working, non-specialist understanding of what a large language model, a
prompt, and a token are. You do not need to know LangChain or LangGraph; this book teaches both
from zero, in Chapters 3 and 4, before using either. Every EvoScientist-specific idea — EvoMemory,
AutoSkills, the sub-agent roster, dangerous mode, the streaming vocabulary — is likewise taught
before it is used. No chapter assumes a term you have not yet met.

## How the book is structured

The whole system is drawn once, as a single master diagram, in Chapter 2 — the three-layer stack
(LangChain → deepagents → EvoScientist) and the path one request takes through it, with every
region of the diagram tagged with the chapter that owns it. Nearly every later chapter opens by
naming the region of that diagram it is about to zoom into, so you always know where you are
relative to the whole. Part II teaches the borrowed foundations (the agent loop, then deepagents'
middleware idiom) before EvoScientist's own code appears in earnest. Parts III–VI walk the engine
outward — assembly and the system prompt, the sub-agent team, the middleware stack, model
providers, the sandbox, tools and configuration — then into the self-evolving core that
differentiates EvoScientist from a merely well-configured agent (EvoMemory and AutoSkills), and
finally into persistence, scheduling, and the many surfaces one engine drives. Part VII covers how
the project is built, tested, and shipped — including its most instructive engineering habit,
testing an LLM agent's real graph without ever calling a real LLM. Part VIII is a single capstone
chapter that traces one real, checked-in research request through every mechanism the book taught,
with short callback boxes pointing back to the chapter that owns each piece.

Scattered through the chapters are short **origin-story boxes** — documented incidents from the
project's real issue tracker, each following the same four beats (background → what happened →
cost → how it got mechanized into the code you're reading). They exist because a surprising amount
of a mature agent framework's design is scar tissue from a specific bad night, and reading the
scar is often faster than reading the design rationale.

This book does not have exercises. Each chapter ends with a **Takeaways** list and a **Sources**
table pointing at the exact files and line ranges the chapter drew from — and the standing rule,
repeated throughout: **when this book and the code disagree, the code wins.**

If you want a map of *where* to start rather than reading straight through, the README's four
named reading routes — a ten-minute overview, a build-an-agent-framework path, a
self-evolving-magic path, and cover-to-cover — will get you there faster. This preface is the
"why"; the README is the "which chapters, in what order."
