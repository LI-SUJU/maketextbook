---
name: make-textbook
description: Analyze a code repository OR a set of blog/paper/article URLs and write a textbook that teaches the subject from big-picture design philosophy down to detail. Use when the user asks to generate a tutorial, textbook, or deep-dive book for a repo, or from one or more article/paper URLs. Requires a repo path or one or more URLs.
argument-hint: "<repo-path | url ...> [output-dir]"
---

# make-textbook — (Repo | URLs) → Textbook workflow

You are orchestrating the production of a **textbook** (not a cheatsheet, not API docs,
not a summary) that teaches a reader to *understand* a subject: its purpose, the ideas
that shape it, the reasoning behind them, and — layer by layer — the detail.

The workflow has two possible **sources of truth**, chosen by what the user hands you:

- **Repo mode** — the subject is a codebase (a local path or a git/GitHub repo URL). The
  book teaches that code, from design philosophy down to line-level implementation.
- **Sources mode** — the subject is a *topic*, seeded by **one or more content URLs**
  (blog posts, papers, articles, docs) that the user wants turned into a book. Here you
  first decide whether the URLs even belong in one book, then research the topic, then
  write a textbook grounded in those sources plus your own cited research.

Both modes converge: they each produce a **dossier** (`_notes/dossier.md`), then share
the same **plan → write → assemble** pipeline (Phases 2–4). The differences live entirely
in Phase 0 (intake) and Phase 1 (survey).

**Phase 3 (writing) is gated: you MUST NOT write any chapter until the user has
explicitly approved the book plan in Phase 2.** In sources mode there is a *second* gate
earlier — the topic-coherence check in Phase 1B — where you must not proceed to research
until the user has confirmed which topic the book is about (when the sources don't cohere
on their own). These negotiations are features, not overhead.

Read `references/pedagogy.md` (in this skill's directory) now. It is the writing contract
every agent in this workflow follows, and it defines the two **grounding modes** (repo =
`path:line` code quotes; sources = `[S#]`/`[R#]` citations) that map onto the two source
modes above. You will pass its absolute path to every agent you spawn.

## Phase 0 — Intake, greeting, and mode confirmation

Your **first turn is a conversation, never silent work**. Before surveying, cloning,
fetching, or spawning anything, you must either (a) show the menu below because no input was
given, or (b) tell the user what you understood and get a one-word confirmation. This is the
front door of the whole workflow — a wrong guess here wastes a lot of downstream effort.

### 0a. No usable argument was given → show the menu and stop

If the invocation has no repo path and no URLs (or the only path given doesn't exist), do
**not** ask a vague "what do you want?". Print this numbered menu, adapted to your own
voice, and stop the turn until the user replies:

> I turn a subject into a **textbook**. I can work from either of two inputs — tell me
> which, and I'll take it from there:
>
> 1. **A code repository** — I'll teach that codebase from its design philosophy down to
>    line-level detail. Reply with a **local path** or a **GitHub/GitLab URL**
>    (e.g. `/path/to/repo` or `https://github.com/org/repo`).
> 2. **One or more articles/papers/blog posts** — I'll research the *topic* they cover and
>    write a textbook about it, grounded in those sources plus cited web research. Reply
>    with **one or more `http(s)` URLs** (any mix of blogs and papers is fine).
>
> You can also add, in either case: how big a book you want (a short **primer**, a
> **standard** textbook, or a **comprehensive** one) and anything you especially want
> covered. If you're not sure, just paste the link(s) and I'll propose a plan for you to
> approve before I write anything.

Keep it this shape — two numbered options, each saying *what to reply with* — so the user
knows exactly how to answer.

### 0b. Input was given → detect the mode, then confirm intent before doing work

Parse the arguments. The **first positional argument suggests the mode**:

- A **local filesystem path** (exists as a directory), or a **git/GitHub-style repo URL**
  (host github.com / gitlab.com / bitbucket.org, or a `*.git` URL) → **repo mode**.
- **One or more `http(s)` URLs** pointing at articles/papers/blog posts/docs (content to
  read, not a repo to clone) → **sources mode**. Collect *all* the URL arguments.
- **A path to an already-generated book** (a directory containing `_notes/plan.md`), **or a
  request to expand / deepen / 出一个更详细的版本 of a book made earlier** → **deepen mode →
  Phase E**. Here there is no new subject to survey: the earlier run's dossier, plan, ledger
  and cached `_sources/` are reused to grow the existing book (typically 精简 → 详细). If the
  user just names a book by title, locate its directory under `$BOOKS/` and confirm.

Then **state what you're about to do and ask the user to confirm** — one short message, no
work yet. Name the subject concretely so a wrong guess is caught immediately:

- Repo mode: "You'd like a textbook that teaches the **`<repo-name>`** repository
  (`<path-or-url>`), from its design down to the implementation — right? Once you confirm
  I'll survey it and come back with a book plan for your approval."
- Sources mode: "You'd like a textbook built from these **N** source(s): `<title/host 1>`,
  `<title/host 2>`, … — I'll read them, work out the common topic, and (unless you object)
  research around it to fill a proper textbook. Confirm and I'll start?"

Handle the two ambiguous cases explicitly in that same confirmation, don't silently pick:

- **A single GitHub/repo URL** could mean "teach this repo" *or* "teach the topic this
  page discusses". Default to repo mode, but say so and offer the alternative:
  "That's a repo, so I'll teach the codebase — or did you mean *make a book about the topic
  it's about*? Say the word." (If it's clearly a specific article *inside* a repo host —
  e.g. a blog post, a single doc page — lean sources mode and confirm that instead.)
- **A mix of a repo and content URLs**, or anything you can't cleanly bucket: ask the one
  question that resolves it rather than guessing.

Only after the user confirms (or answers the clarifying question) do you proceed — repo mode
to Phase 1A, sources mode to Phase 1B. If the user's very first message already made the
intent unmistakable ("write me a book about these two papers: …"), a heavy back-and-forth is
wasted — a single confirming sentence folded into your first substantive reply is enough;
the rule is *never start the expensive work on an unconfirmed guess*, not *always spend a
turn asking*.

### 0c. Resolve the output repository (every book lands in `ai_generated_textbooks`)

Books are **not** written into this project. Every book goes into the user's own GitHub
repository named **`ai_generated_textbooks`** — one repo that collects every book this
skill produces, each as its own top-level directory. Set this up in Phase 0, before any
surveying, so the destination and its permissions are settled once:

1. **Find the GitHub owner:** `gh api user --jq .login` (the user is authenticated via
   `gh`). Call this `<owner>`.
2. **Check the repo exists:** `gh repo view <owner>/ai_generated_textbooks`. If it does
   **not** exist, **ask the user before creating it** — this is an outward action:
   "You don't have an `ai_generated_textbooks` repo yet. Create it on your GitHub to hold
   generated books? (private by default.)" On yes:
   `gh repo create <owner>/ai_generated_textbooks --private --description "Textbooks generated by make-textbook"`.
   Respect a "make it public" request with `--public`. If the user declines, ask where
   else the book should go and use that instead.
3. **Ensure a local working clone** at a **stable path — a sibling of this project**:
   default `<parent-of-project-root>/ai_generated_textbooks/` (i.e. right next to the
   maketextbook project, its own separate repo, not nested inside it). If that directory
   isn't already the clone, create it once:
   `gh repo clone <owner>/ai_generated_textbooks <parent>/ai_generated_textbooks`
   (or `git -C <parent>/ai_generated_textbooks pull` if it already exists, to start from the
   remote's latest). Call this path `$BOOKS`. Because a sibling lives *outside* the project
   root, register it for prompt-free writes in step 0d. If the user gave a different
   location in the `output-dir` argument, use that as `$BOOKS` and register that instead.
4. **The book's output directory** is `$BOOKS/<book-slug>/`. Start with a provisional slug
   (`<repo-basename>-book` in repo mode, `<topic-slug>-book` in sources mode); the final
   name is a Phase 2 negotiation item — rename the directory once settled. If
   `$BOOKS/<book-slug>/` already exists and is non-empty, ask before touching it (it may be
   an earlier run to resume, or a name collision to resolve).

Create the working-notes directory `<output-dir>/_notes/` early; all intermediate
artifacts (source cards, dossier, plan, ledger) live there so the run is inspectable and
resumable. At the end of Phase 4 you commit the book to `$BOOKS` and (confirming the first
time) push it to `<owner>/ai_generated_textbooks` — see Phase 4.

### 0d. Permission setup — do it once, up front, so the run stays quiet

Two directory *styles* touch the filesystem, and each has a clean, prompt-free lever.
Configure them **once in Phase 0** rather than discovering prompts mid-run:

- **The output repo (`$BOOKS`) — needs writes.** As a sibling it sits outside the project
  root, so make writes there prompt-free by adding its absolute path to
  `permissions.additionalDirectories` in `.claude/settings.local.json` (this project's
  settings already register the default sibling path). Because it's its own repo living
  outside this project, nothing extra is needed to keep it out of *this* repo's commits —
  it was never inside it. (If a run ever writes the clone somewhere inside the project root
  instead, git-ignore that path so it isn't accidentally committed here.)
- **The study repo (repo mode only) — needs reads.** Clone remote repos shallowly *inside
  `$BOOKS/<book-slug>/_source/<repo-basename>`* (`git clone --depth 1 <url> …`), which is
  already in scope. A **local** `repo-path` lives outside the project, so register its
  absolute path in `permissions.additionalDirectories` — the directory-scoped lever grants
  reads in exactly one folder, far safer and quieter than command wildcards.
- **Git operations** — `git clone`, `git fetch`, `git add`, `git commit`, `git push`, and
  `gh repo`/`gh api` are the only shell commands this workflow needs; they're pre-approved
  in this project's settings. If a needed one prompts (e.g. `git commit` on a fresh
  machine), add the specific form to `permissions.allow` rather than a broad `Bash(git *)`.
- **Figure downloads** — the `figure-ingest` agent downloads useful images with `curl -L`
  (WebFetch can't return binaries). `Bash(curl -L *)` is pre-approved in this project's
  settings; the agent uses it only to fetch page HTML and image files for figures the plan
  called for.

Everything else the workflow builds from is cached where it's cited from: source text under
`$BOOKS/<book-slug>/_sources/NN-slug.md`, notes under `_notes/`. The book dir's own
`.gitignore` (written in Phase 4) keeps `_source/` and `_sources/` out of the *published*
book — cached third-party text and cloned code shouldn't be republished — while `_notes/`
(plan, dossier, figure manifest) may be kept and **`assets/` is committed** (the downloaded
figures are part of the book, referenced by relative path from the chapters).

### Code / source-inspection convention (you and every agent you spawn)

**In repo mode, read code with the Read / Grep / Glob tools, not with Bash.** Read takes a
line offset+limit for exact ranges (never `awk 'NR>=x'` or `sed -n`), Grep searches
content, Glob finds files. Faster, no prompts on external paths, no one-off allowlist
pollution. Reserve **Bash** for what the tools can't do: `git log` / `git blame` /
`git shortlog` for origin-story numbers and dates, and the `git clone` above.

**In sources mode, fetch web content with WebFetch / WebSearch**, and read the cached
`_sources/*.md` files with the Read tool. State the relevant convention in the prompt of
every agent you spawn.

## Phase 1A — Survey the repo (repo mode)

Goal: a **repo dossier** — the raw material the book plan is built from.

1. Do a quick orientation yourself (top-level `ls`, README, manifest files like
   `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`) to identify the major
   subsystems and the project's rough size and type.
2. Spawn `repo-scout` agents — one per major subsystem/concern, **in parallel, in a single
   message**. Typical cuts for a mid-size repo (adapt to what you saw):
   - purpose, domain, and user-facing behavior (README, docs, examples, CLI/API surface)
   - architecture and control flow (entry points, main loop, module dependency shape)
   - core data structures and domain model
   - external dependencies, frameworks, and the *concepts* they bring in (each a candidate
     for "must be taught before use")
   - build / test / run / deploy machinery
   For a small repo (< ~30 files), 1–2 scouts suffice; for a large monorepo, add scouts
   per package. Tell each scout the repo path, its concern, and that its report becomes
   textbook source material.
3. Merge the scout reports into `<output-dir>/_notes/dossier.md`. Resolve contradictions
   by checking the code yourself. The dossier must include a **concept inventory**: every
   non-obvious concept, tool, term, or technique the repo relies on, each marked "reader
   likely knows / must be taught".
4. Start `<output-dir>/_notes/figures.md`, the **figure manifest**, from any candidate
   figures the scouts flagged — images the repo already ships (`docs/*.png`, architecture
   diagrams, screenshots) that would help a reader. One row each: `F<N> | description | source
   (repo path or URL) | target chapter (TBD) | decision (embed / redraw / text-link, TBD) |
   status`. This is provisional; the plan (Phase 2) decides which figures actually make it in.

Then go to Phase 2. (The dossier's grounding mode is **repo**.)

## Phase 1B — Ingest, cluster, and research the topic (sources mode)

This phase has three steps: ingest the URLs, decide whether they form one coherent book
(a gate), then research the chosen topic into a dossier.

### 1B.1 — Ingest each source

Spawn one `source-ingest` agent **per URL, in parallel, in a single message**. Each agent
fetches its URL, classifies it (paper / blog post / article / docs / other), extracts a
**source card** (title, author(s)/venue, date, type, the 3–7 core claims or contributions,
the key concepts it assumes or introduces, and its apparent topic), and caches the cleaned
full text to `<output-dir>/_sources/NN-slug.md` with a stable `[S<N>]` citation key at the
top. Collect the returned source cards into `<output-dir>/_notes/sources.md`.

If a fetch fails (paywall, JS-only page, dead link), the agent says so; note it and, if the
source is load-bearing, ask the user for an alternative (PDF, mirror, or paste).

### 1B.2 — Topic-coherence gate (do NOT skip; do NOT research until it passes)

Read the source cards and decide: **do these sources belong in ONE book with one big main
theme?** Cluster them by topic. Then:

- **Coherent** (all, or a strong dominant majority, share one theme): state the theme you
  see and how each source contributes to it (foundational / a technique / a case study /
  a counterpoint), and briefly confirm it with the user before researching — a one-line
  "these all orbit *<theme>*; I'll build the book around that — good?" is enough. If one or
  two sources are mild outliers around a clear core, propose folding them in as context or
  setting them aside, and let the user pick.
- **Incoherent** (two or more genuinely unrelated clusters): **stop and report**. Present a
  table — one row per cluster — showing the cluster's topic and **which input URLs belong
  to it**, so the user sees exactly how their inputs split. Say plainly that these don't
  form one book. Then use `AskUserQuestion` to let the user choose how to proceed, with
  options drawn from the actual clusters, e.g.:
  - proceed with cluster A (its topic) — *recommended if one cluster is clearly the biggest*
  - proceed with cluster B (its topic)
  - try an umbrella theme that genuinely spans them (only offer this if a real one exists —
    don't manufacture a fake bridge)
  - treat them as separate Parts of one larger book (more work; offer only if it's sane)

  Do not proceed until the user picks. Record the decision (chosen theme, dropped sources)
  at the top of `_notes/sources.md`.

Present clustering **in the conversation** as a readable table, not "see the file".

### 1B.3 — Research the chosen topic (web-research expansion — on by default)

The seed URLs are the *primary* sources, but a textbook needs more than a synthesis of a
few pages: it needs the prerequisites, the surrounding background, competing views, and
worked examples that make the topic teachable. So:

1. From the source cards on the chosen theme, draft a **concept inventory** for the book:
   every concept the topic rests on, each marked "reader likely knows / must be taught".
   The must-teach items and the gaps between the seed sources define the research agenda.
2. Spawn `researcher` agents — one per research sub-question / must-teach concept /
   gap — **in parallel, in a single message**. Each does deep-research-style web fan-out
   (WebSearch → WebFetch), **adversarially verifies** its key claims against a second
   independent source, and returns a cited findings block with a `[R<N>]` key per source it
   relied on and an explicit confidence note on anything it couldn't corroborate.
3. Scale research to the requested book size (Phase 2) and the **research-scope toggle**:
   default is *expand* (fill every must-teach gap with web research); if the user asked to
   stay **faithful to the URLs only**, skip the web fan-out and build the dossier purely
   from the seed sources — still producing the concept inventory, but marking gaps as
   "out of scope: not covered by the supplied sources" rather than researching them.
4. Merge everything into `<output-dir>/_notes/dossier.md`: the theme and narrative
   possibilities, the concept inventory, the verified findings organized by subtopic, and
   a **bibliography** — a numbered list of every `[S#]` seed source (with its `_sources/`
   path) and every `[R#]` research source (with its URL and title). This bibliography is
   what chapters cite and what the book's References section is built from. Flag any claim
   that survived only one source as "single-source — treat with care".
5. Start `<output-dir>/_notes/figures.md`, the **figure manifest**, from the candidate
   figures the ingest and researcher agents flagged — useful diagrams/plots in a seed source
   or found in research (Wikimedia and open docs are good wells). One row each: `F<N> |
   description | source (page or image URL) | target chapter (TBD) | decision (embed / redraw
   / text-link, TBD) | status`. Provisional; the plan (Phase 2) decides which make it in.

Then go to Phase 2. (The dossier's grounding mode is **sources**.)

## Phase 2 — Book plan (negotiate until approved — the gate)

Draft `<output-dir>/_notes/plan.md` containing:

1. **Audience & prerequisites** — who the reader is, what they're assumed to know.
2. **Language** of the book (English / 中文 / mixed).
3. **Grounding mode** — `repo` or `sources` (from Phase 1); every writing agent is told
   which, so it uses the right citation form.
4. **Narrative arc** — the one-paragraph story the book tells: in repo mode, from "why
   does this project exist and what shaped its design" down to "you could now modify it";
   in sources mode, from "why this topic matters and what question it answers" down to "you
   now understand it well enough to reason about / apply / extend it".
5. **Chapter list** — for each chapter: title, one-paragraph goal, the questions it
   answers, the key material it walks through (files/code in repo mode; sources/findings
   `[S#]`/`[R#]` in sources mode), and the concepts it *introduces* vs. *requires* (from
   earlier chapters).
6. **Concept dependency order** — show that every concept is introduced before any chapter
   uses it. This ordering drives the chapter sequence.
7. **Depth calibration** — which areas get detailed walkthroughs (line-level code in repo
   mode; close reading of a source's argument/derivation in sources mode) vs. an
   architectural/overview treatment, and why.
8. **Size**, **detail level**, and **structure** — see the three negotiation levers below;
   record the chosen size preset, detail level (`concise` / 精简 or `detailed` / 详细), and
   structural pattern here with the resulting chapter count and rough per-chapter length.
9. **Book title and output directory name** — propose both (directory name short and
   kebab-case, e.g. `understanding-<topic>/`); the user decides. Rename the provisional
   directory once settled.
10. **Signature devices** — a sketch of the master diagram (the whole-subject map the
    big-picture chapter presents and later chapters zoom into, regions annotated with
    chapter numbers); whether the book ends with a capstone chapter (a real feature traced
    end-to-end in repo mode; a real problem/case worked through the whole framework in
    sources mode); whether chapters carry exercises (思考题); and the 2–3 named reading
    routes the README will offer. See "Devices that earn their place" in the pedagogy
    contract.
11. **Figures** — from the `_notes/figures.md` manifest, decide which candidate figures
    actually earn a place and assign each to a chapter, marking its **decision**: `embed`
    (download the real image into `assets/figures/` and place it, with source attribution),
    `redraw` (an original Mermaid/ASCII diagram of the idea — often clearer, and the right
    call for figures locked in a PDF), or `text-link` (describe it in prose with a link to
    the original). Prefer drawing our own where a concept can be drawn cleanly; reuse real
    images where seeing the actual thing matters. The `embed` rows become the figure-ingest
    work list at the start of Phase 3.

### The three user levers: size, detail level, and structure

The user explicitly chooses **how big** the book is, **how deep** each chapter goes, and
**how it's structured**. Present them with `AskUserQuestion` (offer the presets below;
"Other" lets them specify exactly). Size and detail are independent axes — size is *breadth*
(how much of the subject, how many chapters), detail is *depth per chapter* (how far each
chapter drills down); a book can be broad-but-concise or narrow-but-deep.

**Size** (calibrate chapter count and per-chapter length; scales research effort in
sources mode too):
- **Primer** — ~3–5 chapters, ~1.5–3k words each. A focused conceptual tour.
- **Standard** — ~6–9 chapters, ~3–5k words each. A proper textbook. *(default)*
- **Comprehensive** — ~10–16 chapters, ~4–6k words each. Full, exhaustive treatment.
- **Custom** — the user names a chapter count and/or total length.

**Detail level (详略)** — how deeply each chapter drills, independent of how many chapters:
- **精简 / Concise** — a fast, readable first pass: the full narrative and every idea, but
  trimmed line-level walkthroughs and sparing use of the optional writing devices. Aims at
  the low end of the size preset's word range. *Explicitly resumable:* because the dossier,
  plan, ledger and cached sources are retained, a concise book can later be **deepened**
  into a detailed one without re-surveying or re-researching (see Phase E).
- **详细 / Detailed** — the full pedagogy contract: rich three-layer treatment, line-level
  code/close-reading walkthroughs, and origin-story / tension / four-beat devices wherever
  the material earns them. *(default)*

Offer concise as the low-commitment on-ramp: "want a **精简版** first — a quick readable book
you can skim, and I can expand any part or the whole thing into a detailed version later? Or
go **详细** straight away?" Record the choice; it is passed to every `chapter-writer`.

**Structure** (the organizing principle of the chapter sequence):
- **Progressive depth** — foundations → mechanisms → advanced/edge, each layer building on
  the last. The safe default; best when the topic has a natural difficulty gradient.
- **Thematic parts** — grouped into Parts by subtopic/cluster. Best when sources mode
  surfaced several sub-themes, or the user chose an umbrella theme spanning clusters.
- **Problem-driven** — each chapter is framed by a driving question/problem it resolves.
  Best for a topic the reader approaches with concrete questions.
- **Source/component-anchored** — one chapter (or Part) per major source (sources mode) or
  major subsystem (repo mode), synthesized rather than summarized. Best for a survey feel
  or a repo with cleanly separable subsystems.

Record the chosen size, detail level, and structure in `plan.md` and let them shape the
chapter list and per-chapter depth.

Then present the plan to the user **in the conversation** (summarize it readably; don't
just say "see the file") and negotiate:

- Use `AskUserQuestion` for the discrete choices (language, audience level, depth, size,
  structure, whether to include exercises); open discussion for the outline itself.
- Revise `plan.md` after every round of feedback and re-present the changed parts.
- Repeat until the user clearly approves (e.g. "看起来不错，开始写吧" / "approved" / "go
  ahead"). Enthusiasm about one aspect is not approval of the whole plan. **Do not start
  Phase 3 without it.** If the user is unavailable mid-negotiation, stop the turn after
  presenting the plan — the plan on disk makes resuming cheap.

## Phase 3 — Write the chapters

Initialize `<output-dir>/_notes/ledger.md`: a running record of, per completed chapter,
(a) concepts introduced and the exact term used for each, (b) notation/metaphor choices,
(c) a 3–5 sentence summary. This is how later chapters stay consistent with earlier ones
and avoid re-explaining or, worse, using an unintroduced term.

**Fetch the figures first.** Create `<output-dir>/assets/figures/` and, for every figure the
plan marked `embed`, spawn a `figure-ingest` agent — in parallel, one per figure — giving it
the figure id, its description and source, the `assets/figures/` path, and the `figures.md` +
`assets/CREDITS.md` paths. Each downloads the real image, looks at it to confirm it matches,
and records its source. Collect the returned figure cards (saved path, alt text, caption,
source) — these are what chapter-writers use to place the figures. If an ingest reports a
figure couldn't be fetched, flip that figure's `figures.md` decision to `redraw` or
`text-link` so the writer handles it without a broken reference. (`redraw`/`text-link`
figures need no ingest — the chapter-writer draws or links them directly.)

Compute writing **waves** from the concept-dependency order in the plan: a chapter may be
written once every chapter it depends on is written. Within a wave, spawn `chapter-writer`
agents in parallel (single message); between waves, update the ledger from each writer's
returned ledger entries before starting the next wave.

Each `chapter-writer` gets: the pedagogy guide path, the **grounding mode** (`repo` or
`sources`), the **detail level** (`concise` / 精简 or `detailed` / 详细, from the plan), the
dossier path, the plan path, the current ledger path, its chapter brief (copied inline from
the plan), the output file path (`<output-dir>/NN-slug.md`), **its chapter's figures** (the
relevant `figures.md` rows plus the matching figure cards — saved path, alt text, caption,
source — for `embed` figures already fetched), and — per mode — the **repo path** (repo mode)
or the **`_sources/` directory + dossier bibliography** (sources mode).
Writers ground every claim in real evidence: repo mode quotes code with `path:line`; sources
mode cites `[S#]`/`[R#]` keyed to the bibliography, rendered as clickable `[[S#]](url)`
links. Neither invents.

After each chapter is written, run a `pedagogy-reviewer` agent on it (told the same
grounding mode *and* detail level, so it spot-checks the right kind of evidence and
calibrates its depth checks — it must not fail a deliberately `concise` chapter for lacking
exhaustive walkthroughs). The reviewer returns a
findings list (violations of the pedagogy contract). If there are findings, send the
chapter back to a `chapter-writer` in revise mode with the findings. One review→revise
round per chapter by default; escalate to the user only if a chapter fails review twice.

Pipeline, don't barrier: a chapter can be under review while the next wave's chapters are
being written, as long as ledger ordering is respected.

## Phase 4 — Assemble the book

Spawn a single `book-editor` agent over the whole output directory (told the grounding
mode) to:
- write `README.md` (title, how-to-read-this-book guide, table of contents with chapter
  one-liners), a preface, and `glossary.md` compiled from first-occurrence definitions;
- in **sources mode**, also compile `references.md` — the full bibliography from the
  dossier (`[S#]` seed sources with `_sources/` paths, `[R#]` research sources with URLs),
  and verify that every `[S#]`/`[R#]` cited in a chapter resolves to an entry;
- enforce terminology consistency and add cross-references ("as we saw in Chapter 3…");
- smooth chapter transitions so the book reads as one narrative, not stapled essays;
- verify **figures**: every embedded `![](assets/figures/…)` points at a file that exists,
  carries alt text and a source line, and is referenced from the prose (no floating figures
  or orphan image files); figure numbers run consistently; `assets/CREDITS.md` lists every
  reused image; and reused figures are also credited in `references.md` (sources mode).

Then verify yourself: every chapter file exists, TOC links resolve, no chapter still
contains reviewer TODO markers, no embedded image path is broken, and (sources mode) no
dangling citation keys.

### Publish to `ai_generated_textbooks`

The book lives in `$BOOKS/<book-slug>/` inside the local clone of the user's
`ai_generated_textbooks` repo. Finish by getting it onto their GitHub:

1. **Write the book's `.gitignore`** (if not already present) at `$BOOKS/<book-slug>/`
   ignoring `_source/` and `_sources/` — cloned third-party code and cached article text
   should not be republished. Do **not** ignore `assets/` — the downloaded figures are part
   of the book and must be committed so the chapters' relative image links resolve. Keep
   `_notes/` unless the user says otherwise (the plan, dossier, and figure manifest make
   future revisions cheap and contain no third-party full text).
2. **Commit** inside the clone, operating on it by path so no `cd` is needed:
   `git -C $BOOKS add <book-slug>` then
   `git -C $BOOKS commit -m "Add <book title> (<repo|N sources>)"`.
3. **Push — confirm the first time.** The user asked for books to live in their GitHub, so
   pushing is the intended end state, but it's an outward action: on the first book, say
   "Book's assembled and committed — push it to `<owner>/ai_generated_textbooks`?" and push
   on yes (`git -C $BOOKS push`). If they've already said "just push from now on", skip the
   ask on later runs.

Finally, tell the user where the book is (the repo, the directory, and the pushed
commit/URL if pushed) and its final structure. Then note the two forward paths the retained
artifacts unlock — keep `_notes/` (and, uncommitted, `_sources/`) to make these cheap:
- if the book was written **concise (精简)**, offer to **deepen it** — the whole book or just
  the chapters they found too thin — into a detailed version, reusing the existing research
  (Phase E); a one-liner like "读完之后想要更详细的版本，随时说，我会基于已有的研究把它加深" is enough.
- either way, `_notes/` can be deleted if they want a clean publish, but deleting it means a
  future deepen/revise re-does the survey or research.

## Phase E — Deepen an existing book (精简 → 详细, or expand coverage)

Entered when the user wants **more** from a book an earlier run produced — most often "the
精简版 was good, now make it detailed", but also "go deeper on chapter 5" or "add a chapter
on X". The whole point is **reuse**: the expensive survey/research already happened and lives
in `_notes/`, so you do not redo it.

1. **Locate the book and its artifacts.** Find the book directory (from the path/title the
   user gave, or under `$BOOKS/`). Read `_notes/plan.md`, `_notes/dossier.md`,
   `_notes/ledger.md`, and (sources mode) the cached `_sources/`. If `_notes/` is missing
   (deleted after publish), you cannot cheaply deepen — tell the user the research artifacts
   are gone and offer to re-run the relevant survey/research phase to rebuild them first.
2. **Confirm the ask and scope.** State what you'll change and get approval — this is a
   smaller gate than Phase 2 but still a gate. Typical scopes: (a) **whole-book deepen** —
   every chapter concise → detailed; (b) **selective deepen** — only named chapters; (c)
   **new chapters** — extend coverage (this may need a little targeted research: spawn
   `researcher`/`repo-scout` for just the new material and merge into the existing dossier,
   not a full re-survey). Update `plan.md`'s detail-level field to `detailed` for the
   affected chapters.
3. **Deepen the chapters.** For each chapter in scope, spawn a `chapter-writer` in **Deepen
   mode** (grow the existing file in place, preserving structure/terminology/citations),
   giving it the same inputs as Phase 3 plus the existing chapter path. If deepening calls
   for new `embed` figures, run `figure-ingest` for them first (as in Phase 3) so the writer
   has the images in hand. Respect ledger order:
   a chapter is deepened against the current ledger, and its updated ledger entry feeds the
   next. Run `pedagogy-reviewer` on each deepened chapter as in Phase 3.
4. **Re-assemble and republish.** Re-run the `book-editor` pass (terminology, cross-refs,
   glossary and — sources mode — `references.md` and clickable-citation integrity now that
   chapters have grown), then commit and push as in Phase 4 with a message naming the change
   (e.g. "Deepen <book title>: 精简 → 详细").

Everything else — grounding mode, citation rules, permission setup — is unchanged from the
original run; only the depth increases.

## Failure handling

- A scout / ingest / researcher / writer returning null or garbage: respawn once with a
  tightened prompt; if it fails again, do that piece yourself.
- A URL that can't be fetched: report it; if load-bearing, ask the user for an alternative
  rather than fabricating its contents.
- Mid-run interruption: every phase persists to `_notes/` (and `_sources/`), so on
  re-invocation check those first and resume from the last completed artifact instead of
  restarting — including resuming *after* a passed topic-coherence gate without re-asking.
