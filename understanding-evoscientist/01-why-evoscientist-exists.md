# Chapter 1 — Why EvoScientist Exists

> **This chapter answers:**
> - What problem shaped EvoScientist, and why can't a single-shot chatbot solve it?
> - What does "self-evolving AI scientist" actually mean — concretely, not as a slogan?
> - What is the design bet — "configure a general agent framework, don't build one" — and why does it matter?
> - What is the six-phase research workflow, and where does it literally live in the code?
> - What do the benchmarks reveal about what the system optimizes for?

You are holding a book about a codebase. Before we open a single file in earnest, this opening chapter does one thing: it makes you understand *why this codebase exists at all*, and it hands you the mental model that every later chapter will fill in. Everything downstream — the middleware onion, the memory graph, the ten chat channels, the persistence war stories — is machinery in service of a single ambition, and if you don't hold that ambition in your head, the machinery will look like a pile of clever tricks instead of a coherent design. So this chapter stays at altitude. There is almost no line-level code here. There is, instead, the problem, the philosophy, and one anchor fact that turns the philosophy into something you can see with your own eyes: the system's stated purpose is not marketing prose in a README, it is *literally the string that gets fed to the language model on every turn*. We will end at that string. First, the problem.

## The problem: research is a grind, not a question

Ask a chatbot "what are the open problems in retrieval-augmented reasoning?" and you get an answer in one shot: the model reads your question, produces text, and stops. That is the shape of almost every large-language-model product you have used. It is a **single-shot** interaction — one prompt in, one response out — and for a huge range of tasks it is exactly right.

Research is not one of those tasks. Doing research is a long, multi-stage grind. You survey a field to learn what is already known. You turn that survey into a plan — candidate ideas, ranked, with experiments that could tell them apart. You write code to run those experiments. The code breaks, so you debug it. The experiments produce numbers, so you analyze them, and the numbers are ambiguous, so you iterate: change one variable, re-run, compare. Eventually you write the results up, and then you re-check the write-up against the original question to make sure you actually answered it. The EvoScientist system prompt names this cycle plainly:

> You help researchers move from question to publishable contribution. That spans the full cycle: surveying a field, generating and ranking ideas, designing and running experiments, drafting papers, and responding to reviews.
> — `EvoScientist/prompts.py:34`

Notice what a single-shot chatbot cannot do here. It cannot *run* the experiment — it can only describe one. It cannot debug code it cannot execute. It cannot compare today's ambiguous result against last week's baseline, because it has no memory of last week. And it cannot get better at your particular research the more you work together, because each conversation starts from the same blank slate as the last. The grind has three properties a one-shot model structurally lacks: it needs to *act on the world* (run code, search the web, read files), it needs to *persist* across a long session and across many sessions, and it rewards *accumulated skill* — the second literature survey should go faster than the first because you learned how to do the first.

The naive fix is "use a bigger, smarter model with a longer prompt." That fails for a reason worth stating precisely, because it is the reason this whole system is shaped the way it is. A longer prompt gives the model more to *read at once*; it does not give the model a way to *do things*, to *remember what it did*, or to *carry a skill from one task to the next*. Those three needs — action, memory, a growing skill set — are not solved by scale. They are solved by architecture. EvoScientist is that architecture.

## Vibe research and the human-on-the-loop stance

Before we describe the architecture, we need two pieces of the project's own vocabulary, because they define what kind of system it is trying to be.

The README opens with a phrase the project has coined: **vibe research** — its brand term for loosely-directed, agent-led exploratory research, the research analogue of "vibe coding," where you give a system a direction and a taste for what "good" looks like and let it explore rather than dictating every step. The tagline states it directly: "EvoScientist aims to harness vibe research by enabling self-evolving AI scientists that autonomously explore, generate insights, and iteratively improve" (`README.md:41`). The word "vibe" is doing real work: it signals that you are not writing a rigid script for the agent to execute. You are setting a direction and trusting the agent to fill in the steps, with you steering rather than driving.

That trust immediately raises a control question: if the agent acts on its own, how much does it ask you first? Here the project takes a deliberate stance, and it names the two options it is choosing between. **Human-in-the-loop** means the human approves every action before it happens — every command, every file write, every web search waits for a click. It is safe but exhausting; it turns you into a permission-granting bottleneck. **Human-on-the-loop** — the stance EvoScientist adopts — means the human reviews *direction* at checkpoints rather than approving every action. The README frames the shift as the product's defining choice: "Moving beyond traditional human-in-the-loop systems, EvoScientist adopts a human-on-the-loop paradigm, where AI acts as a research buddy that co-evolves with human researchers and internalizes scholarly taste and scientific judgment" (`README.md:43`).

Hold both terms. Human-on-the-loop is the *default posture* — the agent takes initiative and you steer. Human-in-the-loop is not discarded; it returns in Chapter 7 as a targeted *safety mechanism*, a pause inserted before genuinely risky actions (like running a shell command that could delete files). The design isn't "trust the agent blindly"; it's "trust it by default, and gate the few actions where a mistake is expensive." For now, remember the slogan the way the system prompt phrases it — you will meet this exact line again when we read the agent's constitution below: the human is "on-the-loop (reviewing direction at checkpoints), not in-the-loop (approving every action)" (`EvoScientist/prompts.py:37`).

## What "self-evolving" actually means

"Self-evolving AI scientist" is the kind of phrase that sounds like a slogan and evaporates on contact. Let us make it concrete, because the whole second half of this book is spent building the two mechanisms that make it literally true.

An ordinary agent — even a very capable one that can run code and search the web — is *amnesiac* between sessions and *fixed* in its abilities. Close the terminal, reopen it tomorrow, and it remembers nothing about your project and knows exactly what it knew yesterday. "Self-evolving" is the claim that this system does two things a fixed agent cannot:

First, it **grows a memory**. EvoScientist keeps a persistent, file-based memory the project calls **EvoMemory** — not a database, but a folder of Markdown files that the system automatically distills from each turn of conversation and links together into a knowledge graph that grows across sessions. When it learns that a particular dataset has a leakage risk, or that a certain library version breaks a build, that lesson becomes a note on disk that the next session can read. You will see exactly how this distillation and linking works in Chapter 11; for now, the intuition is enough: the agent takes notes on itself, forever, and reads them back.

Second, it **grows a skill set**. On top of the memory sits a process the project calls **AutoSkills**: it periodically scans its own accumulated memory for recurring patterns and proposes new reusable *skills* — packaged capabilities — for your review. The feature list describes it in one line: AutoSkills "distills recurring patterns from its own memory into reusable skills on a schedule — proposed for your review via `/autoskills`" (`README.md:126`). Note the last clause. The agent does not silently rewrite its own capabilities; it *proposes*, and a human *approves* — the human-on-the-loop stance made concrete, and a preview of the safety gate in Chapter 12, which owns AutoSkills in full.

So "self-evolving" is not mysticism. It is two specific, inspectable mechanisms — a memory that accumulates and a skill catalog that grows from that memory under human review. The system prompt states the ambition in one sentence: the agent "internalize[s] lessons across these cycles by maintaining persistent memory and growing your toolkit through the EvoSkills ecosystem" (`EvoScientist/prompts.py:34`). EvoMemory and AutoSkills are the two names you should carry forward; their machinery is later chapters' job. Here, only the shape matters: *the system that finishes a task is not quite the same system that started it.*

## The design bet: configure, don't build

Now the most important idea in the entire book — the frame that, once you see it, reorganizes how you read every subsequent chapter.

You might expect a system this ambitious to be built from scratch: a hand-rolled agent loop, custom code to call models and dispatch tools, bespoke plumbing for memory and delegation. It is not. Almost none of the "agent-ness" lives in this repository. EvoScientist made a bet: **don't build a general agent framework — configure one.** The general machinery of *being an agent* — the loop that lets a model call tools, get results, and call more tools; the virtual filesystem it reads and writes; the ability to spin up specialized helper agents — all of that is borrowed from a stack of two libraries beneath it, **LangChain** and, on top of LangChain, a framework called **deepagents**. What EvoScientist itself contributes is *disciplined configuration of that borrowed machinery for one domain*: running scientific research.

You do not need to know how those two libraries work yet — Chapters 3 and 4 teach them from zero. What you need now is the *bet* and why it is smart. Building a robust agent loop is genuinely hard: streaming, retries, tool routing, context-window management, human-approval interrupts, checkpointing so a crashed run can resume. A team that builds all of that from scratch spends its effort on solved problems. A team that *configures* an existing framework spends its effort where it actually has an edge — in this case, the research domain and the self-evolving loop that no general framework provides. The bet buys **leverage** (thousands of lines of hard agent plumbing come for free) and **focus** (the repo's own code concentrates on what makes it EvoScientist rather than a generic assistant).

That bet is the single most useful lens for reading this book. Again and again you will find that a capability you assumed EvoScientist implemented is actually inherited, and the repo's contribution is a thin, opinionated *configuration* on top — a YAML file naming a sub-agent, a small middleware class inserting one behavior, a system-prompt string setting the agent's voice. Chapter 2 draws the full three-layer stack (LangChain → deepagents → EvoScientist, named bottom-to-top by wrapping) as the book's master map. For this chapter, carry the one-sentence version: *the agent is borrowed; the science, the memory, and the skills are EvoScientist's own.*

## The six-phase workflow — and where it literally lives

Here is where the abstraction becomes something you can put your finger on.

EvoScientist structures the research grind into six named phases: **Intake → Plan → Execute & Debug → Evaluate & Iterate → Write Report → Verify**. You can read them as a straight-line process, though as we will see the arrows are softer than they look:

```
Intake ─▶ Plan ─▶ Execute & Debug ─▶ Evaluate & Iterate ─▶ Write Report ─▶ Verify
                        ▲                     │
                        └──── iterate ────────┘
```

- **Intake** — read the request, extract goals, datasets, constraints, and metrics; save the original proposal to `research_request.md`.
- **Plan** — break the work into experiment stages, each with a success signal, tracked in a to-do list.
- **Execute & Debug** — run the experiments (delegating the actual coding), and fix what breaks.
- **Evaluate & Iterate** — compare results against the success signals; if they are weak or ambiguous, change one variable and re-run. This is the loop-back arrow above.
- **Write Report** — write the findings to `final_report.md`, with real sources and no fabricated citations.
- **Verify** — re-read `research_request.md` and confirm the report actually answers the original question.

Now the striking part, the fact that turns this from a diagram in a slide deck into a claim about the code. This workflow is not a picture someone drew for the documentation. It is a string — a block of instructional text — that gets assembled and handed to the language model as its standing instructions on every turn. In LLM terms, that standing block of instructions is the **system prompt**: the text prepended to every model call that tells the model who it is and how to behave. EvoScientist's system prompt is its *constitution*, and the six phases are Steps 1 through 6 of it, written out in `EvoScientist/prompts.py`. You can watch the phases appear as literal section headers in that file: "## Step 4: Evaluate & Iterate" at `EvoScientist/prompts.py:130`, "## Step 5: Write Report" at `EvoScientist/prompts.py:183`, "## Step 6: Verify" at `EvoScientist/prompts.py:189`. The workflow *is* code — or rather, it is prose that the code compiles into the model's instructions.

The function that assembles this constitution is `get_system_prompt()`, and its docstring lists exactly what it concatenates:

```python
def get_system_prompt(
    *,
    dangerous: bool = False,
    cwd: str | None = None,
) -> str:
    """Generate the complete static system prompt.

    Sections are concatenated in this order:

    1. :data:`EVOSCIENTIST_IDENTITY`
    2. :data:`EXPERIMENT_WORKFLOW`
    ...
```
— `EvoScientist/prompts.py:400-411`

We will read this function properly in Chapter 5, where the full four-source layering of the prompt is the subject. For now, one section of it deserves to be quoted whole, because it is the best "read one paragraph, get the philosophy" artifact in the entire repository — the very first section, the agent's declared identity:

```python
EVOSCIENTIST_IDENTITY = """# Identity

You are EvoScientist, a self-evolving AI research scientist. You are not a workflow executor — you are a research collaborator that grows alongside your human partner across sessions.
```
— `EvoScientist/prompts.py:29-31`

Read that first sentence twice. "You are not a workflow executor." A rigid script would *be* a workflow executor — it would march Intake → Plan → Execute in lockstep. But the identity block explicitly disclaims that, and this is why the six phases above are drawn with a soft iteration arrow rather than a rigid pipeline: the same section tells the agent to "take initiative" and "propose the next useful step rather than waiting for micro-instructions," immediately followed by the human-on-the-loop line we met earlier — the human is "on-the-loop (reviewing direction at checkpoints), not in-the-loop (approving every action)" (`EvoScientist/prompts.py:37`). The six-phase workflow is a *recommended structure*, not a cage. The design tension is right there in the text: enough structure that the grind is organized, enough freedom that the agent can behave like a collaborator rather than a form-filler. That tension — structure versus initiative — is resolved not by choosing one but by encoding both into the constitution and trusting the model to balance them. When we reach Chapter 5, you will see that the workflow section even calls its own plan step "flexible, not rigid."

The reason this fact matters so much for the rest of the book is that it demonstrates the "configure, don't build" bet in miniature. EvoScientist did not build a workflow engine with a state machine for the six phases. It *wrote down the workflow as instructions* and let the borrowed agent loop follow them. The system's behavior is steered by prose far more than by control flow — which is exactly why `prompts.py` is the file to read if you want to understand what this agent believes about itself.

## What the benchmarks reveal

A system's benchmark suite is a confession: it tells you what the builders decided was worth being good at. EvoScientist reports top rankings on five external evaluations, and reading the list closely reveals the shape of the system underneath.

The README's changelog dates each result: #1 on **DeepResearch Bench** (18 Apr 2026, `README.md:142`) and on **DeepResearch Bench II** (`README.md:143`), which measure long-horizon literature-style research; #1 on **AstaBench Code & Execution** (25 Mar 2026, `README.md:145`) and #1 on **AstaBench Data Analysis** (26 Mar 2026, `README.md:144`), which measure writing-and-running code and analyzing its output; and #2 overall — #1 among agents built on one particular base model — on **ResearchClawBench** (03 Jun 2026, `README.md:141`). These are not chatbot benchmarks. Not one of them can be won by a single-shot response. Each requires the agent to *act over time*: search, read, run code, iterate, and produce a substantial artifact. The benchmark suite is a direct measurement of exactly the "grind" capabilities we argued a single-shot model structurally lacks.

There is a sharper observation, and it is the one to remember. Look at what the benchmarks measure — deep research, code execution, data analysis — and then look at the agent's internal team. The README describes a "Multi-Agent Team" of "6 sub-agents (plan, research, code, debug, analyze, write)" (`README.md:124`): a main agent that delegates to specialists, one of whom does research, one of whom writes code, one of whom analyzes data. The eval suite mirrors that roster almost one-for-one. *DeepResearch* Bench tests the **research** specialist's domain. AstaBench *Code & Execution* tests the **code** and **debug** specialists. AstaBench *Data Analysis* tests the **analyze** specialist. The org chart is the eval suite. That correspondence is not a coincidence; it is what it looks like when a system is designed around a decomposition of research labor and then measured against benchmarks that isolate each kind of labor. You will meet these six sub-agents (plus a seventh, unattended one — the scheduler) in Chapter 6.

## A session you can hold in your head

Abstractions settle once you have watched one concrete run, so here is a real one, checked into the repository under `docs/examples/survey-literature/` — the same example the book's final capstone (Chapter 17) will trace end-to-end through every mechanism we teach. Seed the picture now.

You give EvoScientist a topic and a single instruction — the actual prompt from the example is: "Use paper-navigator to write a systematic survey of SIGIR 2026 papers publicly available on arXiv. Generate the English version first, then translate it into Chinese. Save both as local .md files" (`docs/examples/survey-literature/README.md:54-58`). That is *one* human message. What the agent does with it is the grind, run autonomously: it decomposes the topic into several variant search queries, discovers candidate papers across arXiv and citation graphs, deduplicates and clusters them into a thematic taxonomy, reads each paper with a structured evaluation, drafts the survey section by section with inline citations, and finally writes the English version and translates it to Chinese (`docs/examples/survey-literature/README.md:15-20`). The checked-in output covers 68 arXiv papers organized into ten sections (`docs/examples/survey-literature/README.md:29`) — a conference-grade artifact, produced from a one-line request while you stayed on the loop.

That is the whole thesis compressed into one session: a long multi-stage task (survey → cluster → read → draft → translate), sustained autonomously by a borrowed agent loop, organized by the six-phase constitution, drawing on an installed skill (`paper-navigator`), under a human who steered the direction and reviewed the result rather than approving each of the 68 paper-reads. Every capability in that sentence is a chapter of this book.

## Where this leaves you

You now have the frame the entire book is built on. Research is a grind that a single-shot model cannot sustain, because the grind needs action, memory, and a growing skill set — none of which come from a bigger prompt. EvoScientist meets that with a specific stance (vibe research under a human-on-the-loop paradigm), a specific pair of mechanisms that make "self-evolving" literal (EvoMemory and AutoSkills), and a specific engineering bet (configure a borrowed agent framework rather than build one) that concentrates the team's effort where it matters. And you have seen the anchor fact that keeps all of this honest: the six-phase workflow and the agent's very identity are not documentation — they are strings in `prompts.py`, compiled into the model's instructions on every turn.

Chapter 2 turns this frame into a picture. It presents the book's single master diagram — the three-layer stack from LangChain up through deepagents to EvoScientist, with a request flowing through it — and every region of that diagram is tagged with the chapter that explains it. From here on, each chapter opens by naming the region of the map it zooms into. You have the *why*; next comes the *where*.

## Takeaways

- **Research is a grind, not a question.** Survey → plan → execute → debug → analyze → write → verify is a long, multi-stage loop that a single-shot chatbot cannot sustain, because it needs to *act*, to *remember*, and to *accumulate skill* — none of which a longer prompt provides.
- **Vibe research + human-on-the-loop** is the stance: give the agent a direction and taste, let it explore, and review its *direction* at checkpoints rather than approving every action. (Human-in-the-loop returns in Chapter 7 as a targeted safety gate.)
- **"Self-evolving" is two concrete mechanisms**, not a slogan: **EvoMemory** (a growing, file-based knowledge graph distilled from each turn — Chapter 11) and **AutoSkills** (new reusable skills mined from that memory, proposed for human approval — Chapter 12).
- **The design bet is "configure, don't build."** Almost none of the agent machinery lives in this repo; EvoScientist configures the LangChain → deepagents stack for one domain, buying leverage and focus. This lens reorganizes how you read every later chapter.
- **The six-phase workflow — Intake → Plan → Execute & Debug → Evaluate & Iterate → Write Report → Verify — literally is code**: it is the system prompt assembled by `get_system_prompt()` in `prompts.py`, a recommended structure rather than a rigid pipeline.
- **The benchmark suite mirrors the sub-agent roster.** Top rankings on deep-research, code-execution, and data-analysis benchmarks map one-to-one onto the research, code, and analyze sub-agents: the org chart is the eval suite.

## Sources

| Topic | Authoritative file(s) |
|---|---|
| Vibe research, human-on-the-loop stance, feature list | `README.md` (tagline `:41-43`; features `:123-134`) |
| Benchmark results and dates | `README.md` (awards `:45-121`; changelog `:141-145`) |
| The agent's identity and operating principles | `EvoScientist/prompts.py` (`EVOSCIENTIST_IDENTITY`, `:29-41`) |
| The six-phase workflow (Intake → … → Verify) | `EvoScientist/prompts.py` (Steps 1–6, incl. `:130`, `:183`, `:189`) |
| System-prompt assembly (`get_system_prompt`) | `EvoScientist/prompts.py:400-445` |
| The concrete literature-survey session | `docs/examples/survey-literature/README.md` |
| One-sentence thesis; "configure, don't build"; three-layer stack | Book dossier §0–§1 (grounded in `EvoScientist/EvoScientist.py`, expanded in Chapter 2) |

When the book and the code disagree, **the code wins** — this repository is the law, and the book is a guide to reading it.
