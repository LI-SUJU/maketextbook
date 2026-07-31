# Pedagogy contract — how this textbook is written

Every agent that writes or reviews prose for this book follows this contract. When a
rule here conflicts with brevity, the rule wins: this is a textbook, and a reader's
understanding is the only success metric.

## 1. Big picture before mechanism（先高层，后细节）

The book — and every chapter — starts from *why*. Before showing how something works,
establish: what problem it solves, why the problem is hard, what the designers were
optimizing for, and what alternatives they rejected (and why). The reader should come
away understanding the project's 设计/开发/思考 philosophy, not just its behavior.
Design rationale is first-class content: if the code chose a queue over direct calls,
the chapter explains the tradeoff, even if you must infer it from the code and say
you're inferring.

## 2. 深入浅出，粗中有细 — layered depth

Explain in layers, each one complete at its own altitude:

1. **Intuition** — a plain-language account, an analogy or a concrete scenario a reader
   can hold in their head.
2. **Mechanism** — the actual moving parts and how they interact, with a diagram
   (Mermaid or ASCII) where structure matters.
3. **Detail** — real code from the repo, quoted with `path:line` references, walked
   through in narrated chunks: a few lines of code, then a paragraph of prose about
   what those lines do *and why they're written that way*, then the next chunk.

Never skip layer 1 to get to layer 3 faster. Never stay at layer 1 when the chapter's
job is depth — the "粗" gives orientation, the "细" gives real understanding, and a
chapter needs both.

## 3. Teach the concept before using it（先解释，后运用）

Before showing *how the repo uses* a concept, tool, library, or technique, first teach
*the thing itself* — what it is, what problem it exists to solve, and enough of a mental
model that a reader with no prior exposure builds intuition for it. Only then show how
this project applies it. Assume the reader may have zero intuition for it.

Concretely: a chapter that walks through the project's Redis caching layer first spends
a section on what a cache is, what Redis is, and why in-memory key-value stores are fast
— *then* reads the project's caching code. A dependency-ordered book plan exists
precisely so this rule never forces a forward reference.

## 4. Define terms at first occurrence（生僻词首次出现即解释）

The first time any jargon, acronym, or project-specific name appears, define it
immediately in the sentence or the one following — a reader must never hit a term and
have to search for its meaning. Each first-occurrence definition is also a glossary
entry (the book editor compiles these). If a term was defined in an earlier chapter,
don't redefine it — a brief parenthetical reminder or a "see Chapter N" reference is
right. The ledger tells you which terms are already introduced; consult it, and never
use a term no earlier chapter (or your own chapter, earlier on the page) has defined.

## 5. Textbook, not cheatsheet（是教科书，不是速查表）

Write in flowing, connected prose. Paragraphs carry the argument; transitions carry the
reader between ideas. Bullet lists are allowed only for genuinely enumerable items
(a list of config keys, a set of states) — never as a substitute for explanation.
Each chapter has:

- an **opening** that says where we are in the book's arc, what this chapter answers,
  and why the reader should care;
- a **body** that develops ideas in an order chosen for the reader (not the order of
  files on disk);
- a **closing** that consolidates: what the reader now understands, how it connects
  forward.

Prefer one example developed deeply over five mentioned shallowly. Ask of every
paragraph: "does this build understanding, or merely record a fact?" — facts without
scaffolding belong in a reference manual, not here.

## 6. Ground everything in real evidence

Every claim in the book must be traceable to a real source — never invent, and never
present a guess as a fact. This workflow has two **grounding modes**; the orchestrator
tells each agent which one is in force, and it decides the citation form:

- **Repo mode** (the book teaches a codebase). Evidence is code and docs actually in the
  repo. Quote real code with `path:line`; never invent or 'simplify' code that doesn't
  exist (clearly-labeled illustrative pseudocode for a *general* concept is fine).
- **Sources mode** (the book teaches a topic drawn from URLs + research). Evidence is the
  fetched source documents in `_notes/../_sources/` and the verified findings in the
  dossier. Attribute every non-obvious claim with an inline citation keyed to the
  dossier's bibliography — `[S3]` for a seed source, `[R7]` for a research finding — so a
  reader can trace any statement back to the page it came from. **Make every citation
  clickable** (see "Clickable citations" below): if the source has a URL, the inline key
  links to it. Quote a source's own words when the wording matters; paraphrase otherwise,
  but still cite. Never state a claim the dossier does not support, and prefer a primary
  source (the paper, the original blog) over a secondary mention of it.

### Clickable citations（引用即链接）

External sources are for *following*, not just for crediting — a reader who wants the
original should be one click away. So citations are rendered as real Markdown links, never
bare brackets:

- **Inline, sources mode.** Write each key as a link to the source's URL:
  `[[S3]](https://the-source-url)` renders as a clickable `[S3]`. `[R7]` links to that
  research source's URL the same way. If a source genuinely has no URL (e.g. text the user
  pasted, or a paywalled item with no stable landing page), leave the key bare (`[S3]`) — it
  still resolves in the References section, which is the one place that must list every key.
  The dossier bibliography carries the exact URL for every `[S#]`/`[R#]`; use it verbatim, do
  not reconstruct URLs from memory.
- **The References section** (`references.md`, built by the editor) lists every source as a
  clickable entry — the title itself is the link: `[S3] [Paper title](https://…) — author,
  venue, date`. This is the canonical, always-present target, so even a bare inline key
  leads the reader somewhere.
- **Repo mode.** The primary reference stays `path:line` (the code is the authority). When
  the repo has a known public host, you *may* additionally link a quote to its blob URL
  (`[src/app.py:42](https://github.com/org/repo/blob/<sha>/src/app.py#L42)`); keep the bare
  `path:line` as the visible anchor so the reference survives even without the link.

In either mode, when you infer intent that the evidence doesn't state outright, say so
("the code suggests…", "the authors presumably chose this because…", "no source states
this directly, but…"). An honest inference is welcome; a disguised one corrodes the whole
book.

## 7. Chapter anatomy（每章的固定骨架）

Every chapter carries this skeleton (the flesh varies):

- **Question-driven opening.** The chapter starts with a short quoted block — "本章回答以下
  问题 / This chapter answers:" — listing 3–5 concrete questions. This is the chapter's
  contract with the reader; every question must actually be answered by the end. Follow
  it with a paragraph situating the chapter in the book's arc.
- **Closing consolidation**: a 要点/takeaways list that compresses the chapter (this is
  the one place bullet points are the right form); 思考题/exercises *if the plan opted
  in* — transfer questions ("如果让你来设计…") with hints, not recall quizzes; and a
  **sources table** mapping each topic taught to its authority — the file(s) in the repo
  (`path` level) in repo mode, or the cited source/finding (`[S#]`/`[R#]`, rendered as the
  clickable links defined above) in sources mode. The book is a guide, the evidence is the
  law: state explicitly that when book and
  evidence disagree, the evidence wins — the code in repo mode, the cited source in
  sources mode. This keeps the book honest as the material evolves.

## 8. Devices that earn their place（值得用的写作装置）

Use these where the material supports them; never fake the material to use the device.

- **One master diagram.** The big-picture chapter contains a single whole-system diagram,
  each region annotated with the chapter that covers it（图上直接标章号）. Every later
  chapter opens by saying which region of the map it zooms into. Choose the diagram's
  reading direction for causality, and say so ("read bottom-up: each layer's constraints
  shape the layer above").
- **Origin stories（事故档案）.** When git history, code comments, issues, or design docs
  reveal *why* a mechanism exists — especially a failure that motivated it — teach the
  mechanism through that story, in a quoted box: 背景 → 经过 → 代价 → 机制化. A rule with
  its origin story is remembered; a bare rule is forgotten. Only documented incidents;
  an inferred motivation is presented as inference, never dressed up as history.
- **Intuition → precise statement → embodiments → violation.** For a design principle,
  this four-beat micro-structure（直觉 → 精确表述 → 它在系统里的化身 → 违反会发生什么）
  turns an abstraction into something concrete twice over: first in real mechanisms,
  then in the failure you'd get without it.
- **Tension sections（张力）.** When two design goals genuinely conflict, show the
  conflict explicitly and how the design resolves it. Tradeoffs are the deepest layer of
  understanding — a book that presents every decision as obviously right teaches less
  than one that shows what each decision cost.
- **Concrete numbers over adjectives.** "回填率 3/70" convicts a design; "rarely filled
  in" merely gossips about it. Mine the repo (git log, counts, dates) for real numbers.
- **Capstone case study.** If the plan includes one, the final chapter traces a single
  real feature/request/PR end-to-end through every mechanism the book taught, with
  「🔍 机制回看」callback boxes linking each scene to the chapter that owns it. It
  converts ten chapters of parts into one running machine.

## 9. Figures and images（图表）

A figure is worth its space only when it *builds understanding faster than prose would* — a
structure, a flow, a comparison the eye grasps at once. Decorative images earn nothing; a
chapter with no good figure is better than one padded with a stock illustration. Ask of every
figure the same question as every paragraph: does this teach?

There are three kinds of figure, in rough order of preference:

1. **Drawn by us** — Mermaid (flowchart, sequence, state, ER, class…) or ASCII where
   structure or flow matters, and Markdown tables for enumerable/comparative data. These are
   the backbone: text, versioned, consistent with the book's visual language, and rendered
   natively on GitHub. When a concept can be drawn cleanly, **draw it** rather than hunting
   for someone else's picture — an original diagram is usually clearer and always fits.
2. **Reused from a real source** — a genuinely useful figure from the studied repo, a seed
   source, or web research (Wikimedia and openly-published docs are rich wells). The policy
   is simple: **if it's key and useful, the workflow downloads it into the book's
   `assets/figures/` and embeds it locally, and you always attribute its source.** (The book
   is for the user's own use in a private repo, so there is no licensing gate to apply here —
   but attribution is not optional: every reused figure names where it came from.)
3. **A source's figure we cannot or should not reproduce** — when a figure lives only inside
   a PDF we can't extract, or a redraw would simply be clearer: **redraw the idea** as an
   original Mermaid/ASCII diagram, or describe it in prose with a link to the original
   ("作者用一张时序图展示了…（见原文 [Figure 3](url)）"). Never fabricate an image, and never
   describe a figure you (or the figure-ingest agent) did not actually look at.

**How figures appear on the page.** Markdown has no native `figure`/`caption`, so use this
portable convention, every time:

- Embed a local image with alt text and a caption line beneath it:

      ![alt text describing the image](assets/figures/F3-arch.png)
      *图 3-1：<what it shows / what it teaches>。来源：[<author or site>](url)*

- **Alt text is mandatory** — it serves screen readers and shows when an image fails to load.
- **Every figure carries a source line** (author/site + link for reused figures; "原创图" for
  ones we drew). Reused figures are also listed in `assets/CREDITS.md` and the References
  section.
- **Number figures `图 章-序`** (or `Figure N-M`) and **reference every figure from the prose**
  — "如图 3-1 所示…". A figure the text never points at is floating; either wire it in or cut
  it. Prefer local embeds (`assets/…`) over hot-linked remote URLs so the book survives
  offline and PDF export.

Everything a figure asserts must be traceable the same way any other claim is (§6): its
source is its authority, and when a reused figure and the current code/source disagree, the
evidence wins.

## Voice

Address the reader directly ("you"), present tense, confident but honest about
uncertainty. Zero filler ("in this section we will…" is fine once as an opening move,
not as padding). Humor is welcome when it serves memory; cleverness that obscures is
not. If the book's language is Chinese, keep established English technical terms in
English with a Chinese gloss at first occurrence (e.g. "事件循环（event loop）").
