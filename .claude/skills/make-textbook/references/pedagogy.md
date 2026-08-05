# Pedagogy contract — how this textbook is written

Every agent that writes or reviews prose for this book follows this contract. When a
rule here conflicts with brevity, the rule wins: this is a textbook, and a reader's
understanding is the only success metric.

## 0. Who we write for: a beginner we grow into an expert（读者心智模型：把初学者一步步带成高手）

Hold one mental model of the reader for the entire book: **a motivated beginner** — smart and
willing, but new to this subject, not yet knowing its vocabulary, tools, or mental models. Two
commitments follow, and they **govern every other rule below**:

1. **Guarantee a beginner can follow every step.** Understanding may never depend on something
   the book hasn't explained yet. A reader who has read *up to here* must be able to follow the
   *next* paragraph. If a sentence would lose that reader, it is a bug — slow down, add the
   missing rung, give an example. Comprehension is checked *forward*: each concept must be clear
   enough that the concepts built on it can be clear too.
2. **Progressively turn that beginner into an advanced reader.** The book is a staircase, not a
   plateau. Start each thread at the beginner's level and deepen it in deliberate stages
   (intuition → mechanism → precise detail, §2; and easy case → complications → frontier) so
   that the same person who opened as a novice can, by the end, reason like an expert. Never
   open at the expert's altitude and strand the beginner; never stay at the beginner's altitude
   and starve the advanced reader. Climb — one secure rung at a time.

**The floor is explicit, and it is written down.** "Beginner" never means "knows nothing": the
book plan states, as a short named list, exactly what the reader is assumed to already know —
the **assumed-known floor**（读者已知底线）, e.g. "high-school algebra; can read Python" or
"linear algebra and gradients". Everything on that list may be used without a full lesson;
everything above it must be taught before use, *no matter how routine it feels to an expert*.
Two consequences:

- A mathematical operation or background idea **above the floor** (transpose, dot product,
  softmax, eigenvalue, variance, …) is itself a load-bearing concept and gets the full §4
  treatment. A writer can hold the beginner-first standard perfectly for the subject's own
  ideas while silently assuming a math course — that is exactly the failure this rule exists
  to prevent.
- Floor items are assumed *rusty*, not fresh: where a floor item does load-bearing work at a
  specific spot, give it a **refresher box** (§4's second register) there, not a full beat.

The floor is stated once for the reader (preface/README) and handed verbatim to every writing
and reviewing agent; every "could the beginner follow this?" check is asked *relative to it*.

Everything below — big-picture-first, layered depth, explain-before-define, concept-before-use —
is machinery for these two commitments. When in doubt, ask: *"Could the beginner I'm growing
follow this — and did it move them one rung up?"*

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

1. **Intuition** — a plain-language account **plus at least one concrete example** (a
   specific case, small numbers, or an analogy made concrete) the reader can hold in their
   head. The example is mandatory for any concept a newcomer could stumble on — it is the
   thing that makes the idea stick, not a garnish. An abstract statement with no example is
   an unfinished intuition layer.
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

## 4. Explain a concept the first time — don't just define it（首次出现要"讲透"，不是"一句带过"）

Defining a term in one dense sentence is **not** teaching it. The single most common way this
book fails a reader is a first-occurrence "definition" that stacks several unfamiliar ideas into
one clause: the reader stalls on that sentence, and because everything after it depends on it,
the rest of the section collapses too. A murky prerequisite doesn't stay local — it cascades.
Prevent this with a hard rule.

**Every load-bearing concept, the first time it appears, gets its own explanatory beat — not a
buried inline clause.** The beat runs in this order:

1. **Plain-language intuition** — say what it is using only words a newcomer already knows. No
   new jargon in this move.
2. **A concrete example you can hold in your head** — a specific small case, tiny numbers, or a
   familiar situation made concrete. Every non-obvious concept gets one; "abstract statement,
   no example" is a defect, not a style choice.
3. **The precise definition**, and only then **how this book/repo uses it**.

Rules that make the beat land:

- **One new idea per sentence.** Never introduce a term by leaning on two or three *other* terms
  the reader also doesn't know. If a concept is best understood by contrast ("X is not A, it's
  B"), give A and B each its own sentence and its own concrete image — don't compress the
  contrast into one line.
- **The most load-bearing concepts get their own short sub-section with a heading**, not a
  sentence — especially prerequisites later chapters build on. Clarity compounds: make the
  foundational concept crystal clear and everything downstream can be clear too.
- **Analogies must be self-contained（比喻必须自明）.** An analogy teaches only if the reader
  can follow the analogy's *own story* with zero technical knowledge — an everyday experience
  they have actually had (money, queues, cooking), not a technical construction in costume.
  Never build the analogy out of the very concept it exists to explain ("spin the compass rose
  and relabel the headings" as an analogy for changing coordinates is circular — following it
  already requires the concept). Prefer one analogy developed deeply and then mapped
  element-by-element onto the technical content ("account = slot, balance = activation, the
  midnight overdraft rule = ReLU") over three gestured at. Test: told to someone with no
  background at all, would the story itself land? If an analogy needs its own explanation, it
  is a liability — cut it and say the plain thing.
- **The restate test.** After the beat, ask: could a reader who did *not* already know this
  concept now restate it in their own words and reproduce the example? If not, the beat is too
  terse — expand it. Write for *that* reader, never for one who already gets it.
- Minor/peripheral terms a reader surely knows can still be handled in a quick inline gloss;
  reserve the full beat for concepts that are non-obvious or load-bearing. Every first-occurrence
  explanation is also a glossary entry. If a term was defined earlier, don't re-teach it — a
  brief reminder + "see Chapter N" is right (consult the ledger; never use a term no earlier
  chapter or earlier passage has explained).

**Worked example of the standard — this difference is the whole point:**

*Too terse (a real failure — do NOT write like this):*
> "A distributed representation is the idea that a concept is represented not by a single
> dedicated unit ('grandmother cell') but by a pattern of activity spread across many units, and
> conversely that each unit participates in representing many concepts."

That one sentence makes the reader swallow *grandmother cell*, *pattern across units*, and *each
unit in many concepts* all at once, with no example. Rewrite it as a beat:

*Right (a dedicated beat, one idea at a time, with a concrete example):*
> **How does a network store one idea?** Picture a layer of just three neurons and ask how it
> could represent the concept "dog."
> - **One option — a dedicated neuron.** Neuron 1 fires *only* for "dog," nothing else. People
>   nickname this a "grandmother cell": as if one single neuron lit up only when you saw your
>   grandmother. Simple, but wasteful — three neurons could then store only three concepts.
> - **The other option — a pattern.** "Dog" is stored as a *combination* across all three: say
>   neuron 1 fully on, neuron 2 half on, neuron 3 off — the pattern (1.0, 0.5, 0). No single
>   neuron *is* "the dog neuron"; the *pattern* is.
>
> This second option is a **distributed representation**. Two consequences follow, and we'll use
> both. First, one concept is spread over many neurons. Second — the mirror image — one neuron
> helps represent many concepts: neuron 2 might be half-on for "dog," fully on for "wolf," a
> third on for "loyal." So you can't read a network one neuron at a time and expect each to mean
> one thing. *(That is the seed of superposition, Chapter 4.)*

The good version uses a question, a concrete 3-neuron example with real numbers, one option per
bullet, names the term only *after* the picture exists, then unpacks the consequences one at a
time. **That is the bar for every load-bearing concept in the book.**

### The four registers — and when each is legal（四个档位）

Every technical term's first occurrence is handled in exactly one of four registers. Choosing
the register is an editorial decision, not a shortcut:

1. **Full beat** (above) — for any non-obvious, load-bearing concept this chapter owns.
2. **Refresher box** — for a concept on the assumed-known floor (§0) that does load-bearing
   work *here*: a short quoted box, 1–3 sentences, that reactivates rather than teaches, and
   ends by cashing the concept into this chapter's use of it (the "so…" clause):

   > **Refresher — eigenvalue:** a matrix's eigenvector is a direction it doesn't rotate, only
   > stretches; the stretch factor is the eigenvalue. So an OV eigenvalue near +1 means the
   > head writes back almost exactly what it read — it copies.

   Twenty seconds for the prepared reader, a saved section for the rusty one.
3. **Inline gloss** — one clause in passing, only for minor/peripheral terms that carry no
   weight later.
4. **Forward promise** — the *only* legal way to mention a concept a later chapter owns:
   "Chapter N develops X in full; for now, read it as ⟨one-clause placeholder intuition⟩."
   A later-chapter term used early without its promise is a defect: it tells the reader they
   missed a rung that was never built. (Consult the plan's concept-ownership: owned downstream
   → wear the promise; owned upstream → a one-clause reminder + "see Chapter N".)

**Bold marks definitions, nothing else.** Bold a term exactly where the book defines it (beat,
refresher, or gloss) and never for emphasis — then "bolded somewhere above this line" and
"safe to use" mean the same thing, for the reader and for the reviewer. Use italics for
emphasis.

**New-concept density budget.** A section that introduces more than ~5 load-bearing concepts —
or a chapter beyond ~12 — is overloaded even if every individual beat is perfect: cognitive
load compounds. Either split the material, or (for a genuine survey/field-guide chapter the
plan calls for) *declare the register to the reader* in the opening ("this chapter is a field
guide — hold the shape, not every name") and consolidate with a mid-chapter summary table.
The reader must never discover halfway through that the rules changed.

## 4a. Mathematics: an equation is a concept（公式也要讲透）

Everything §4 demands for concepts applies to mathematics, where the failure mode is quieter:
a formula can be *stated* in a way that looks like teaching but locks the beginner out. Hard
rules:

- **The equation beat.** Every displayed equation, loss, or formal definition gets:
  (a) every symbol defined at or before first use; (b) a term-by-term prose walk — what each
  piece is, what it does, why it is there; (c) a **small-numbers instantiation** the reader
  can verify by hand ("suppose x = (1, 0) and W = [1, −1]: then h = 1·1 + (−1)·0 = 1 …").
  An equation stated but never instantiated is an unfinished beat.
- **The compute test** (the restate test for math): could the reader now evaluate this
  equation on a 2–3-number example using only what the book has given them? If no passage
  enables that, the equation is not yet taught.
- **One running instance across the layers.** Carry a single tiny example from the intuition
  layer into the formal layer, so the reader watches the formula reproduce the case they
  already believe. Never let the analogy and the math live in separate worlds — if the
  intuition was "interference is rare when features are sparse" and the formula is `WᵀW`,
  push the same two-feature example through `WᵀW` on the page.
- **Notation is vocabulary.** The first use of any symbol or notational convention — `ᵀ`,
  `‖·‖`, a hat, `Σ`, `⁻¹`, `∈ ℝⁿ` — gets a gloss, exactly like a term. A math-heavy chapter
  carries a small **symbol table** (at first use, or as a boxed table the chapter points to),
  and the book's glossary compiles a notation section. Never point the reader at internal
  artifacts ("the ledger's notation") — the reader cannot see them.
- **No silently skipped steps.** "It follows that", "one can show", "the arithmetic works
  out" are banned unless the showing is on the page or the skip is explicitly marked and
  safe ("optional: verify that …"). A derivation either shows every rung or names the ones
  it omits.
- **State the point before the proof（先讲要证什么，再动手证）.** A derivation is not taught
  by its algebra. Before the manipulation: say in plain words what question it answers and
  what each possible outcome would mean. After it: cash the result back into that question in
  one sentence. Then apply the **so-what test** (the compute test's sibling): could the reader
  now state, in one sentence, what was just established and why it matters? Algebra whose
  steps all verify but whose point the reader cannot state is a *magician's proof* — it
  convinces without teaching, and a reviewer who only checks the steps will wrongly pass it.
  And if the argument uses a *move* the reader has never seen — a symmetry/invariance test, an
  exchange argument, a proof by compensation — the move itself is a concept: name it and
  explain how the trick works before performing it.
- **Numbers carry their meaning.** Every reported quantitative result says, in words the
  reader has, what it measures, what its scale is, and what a good vs. bad value looks like.
  "Recovers 79% of the loss reduction" means nothing until the reader knows 79% *of what,
  measured how*.

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
- **Recap and prerequisites.** Next, a short recap move reactivating the earlier-chapter
  concepts this chapter leans on — one clause each plus a chapter pointer, never a re-teach —
  and a one-line "**Builds on:**" list naming those prerequisites explicitly, so a reader
  arriving by a shortcut route knows exactly what to backfill. Backward references must be
  *true*: never write "as Chapter N showed" unless Chapter N actually taught it (check the
  ledger). A false "you already know this" is worse than no reference — the reader concludes
  they forgot a rung that was never built.
- **Closing consolidation**: a 要点/takeaways list that compresses the chapter (this is
  the one place bullet points are the right form); a **capability sentence** — one line
  naming what the reader can now *do* that they couldn't before ("you can now read an SAE
  loss and say what each term buys"), so the book's staircase is visible rung by rung;
  思考题/exercises *if the plan opted
  in* — transfer questions ("如果让你来设计…") with hints, not recall quizzes; and a
  **sources table** mapping each topic taught to its authority — the file(s) in the repo
  (`path` level) in repo mode, or the cited source/finding (`[S#]`/`[R#]`, rendered as the
  clickable links defined above) in sources mode. The book is a guide, the evidence is the
  law: state explicitly that when book and
  evidence disagree, the evidence wins — the code in repo mode, the cited source in
  sources mode. This keeps the book honest as the material evolves.
- **Forward handoff.** Every chapter except the last closes by naming what question remains
  and which chapter takes it up. When the *next* chapter deliberately drops in difficulty
  (a new Part restarting from a lower base), say so — "the next Part needs only the toolkit
  of Chapter 5; the climb restarts lower". A signposted descent reads as a designed breather;
  an unsignposted one reads as a lurch. And on reuse: a concept from an earlier chapter gets
  at most one clause plus the chapter pointer — anything longer must announce itself as a
  deliberate restatement ("worth stating once more, in the language we can now use").

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

**No production leaks.** Reader-facing prose never references the book's production
apparatus — the plan, the dossier, the ledger, agent names, or bracketed research notes
("[R26 context; …]"). Those live in `_notes/`. If a sentence needs their content, say it in
the book's own voice ("this book fixes the notation…", not "the ledger fixes the notation…").
