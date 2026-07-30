---
name: make-tutorial
description: Analyze a code repository and write a textbook that teaches the project from big-picture design philosophy down to implementation details. Use when the user asks to generate a tutorial, textbook, or deep-dive book for a repo. Requires a repo path as argument.
argument-hint: <repo-path> [output-dir]
---

# make-tutorial — Repo → Textbook workflow

You are orchestrating the production of a **textbook** (not a cheatsheet, not API docs)
that teaches a reader to understand a codebase: its purpose, its design philosophy, the
reasoning behind its architecture, and — layer by layer — its implementation details.

The workflow has four phases. **Phase 3 (writing) is gated: you MUST NOT write any
chapter until the user has explicitly approved the book plan in Phase 2.** The plan is
negotiated with the user, possibly over several rounds — that negotiation is a feature,
not overhead.

Read `references/pedagogy.md` (in this skill's directory) now. It is the writing
contract every agent in this workflow follows. You will pass its absolute path to every
agent you spawn.

## Phase 0 — Intake

Parse the arguments:
- `repo-path` (required): the repository to teach. It may be a **local path** or a
  **GitHub/remote URL**. If missing (or a local path that doesn't exist), ask the user
  for it and stop until you have it.
- `output-dir` (optional): where the book goes. Default: a new directory **inside the
  current working directory** (the maketutorial project folder — never inside the target
  repo). Its final name is a Phase 2 negotiation item, so start with the provisional
  name `<repo-basename>-book/` and rename it once the user settles on a name. If the
  target directory already exists and is non-empty, ask the user before touching it.

Create the working-notes directory `<output-dir>/_notes/` early; all intermediate
artifacts (dossier, plan, ledger) live there so the run is inspectable and resumable.

### Where the study repo lives (do this before Phase 1 — it prevents a storm of prompts)

The subject of the book is usually a *different* repo than this project. Every shell
command that touches an absolute path **outside the project root** triggers a permission
prompt, so keep the study repo inside the project and tell the tools it's in scope:

1. **Clone remote URLs into a project-local, git-ignored source dir** — not the session
   scratchpad (whose path changes every run and sits outside the project root). Use a
   stable path like `<output-dir>/_source/<repo-basename>`:
   ```
   git clone --depth 1 <url> <output-dir>/_source/<repo-basename>
   ```
   For a local `repo-path`, you may study it in place. If you need dependency/framework
   source too (to quote framework internals), clone or `pip download`+unpack it under
   `<output-dir>/_source/` as well.
2. **Git-ignore the source dir** so it never lands in commits: append `_source/` (and
   `_notes/` if you don't want it committed) to the project's `.gitignore`.
3. **Register the study repo as an allowed directory** so reads there don't prompt. Add
   its absolute path to `permissions.additionalDirectories` in
   `.claude/settings.local.json` (this is the *directory-scoped* permission lever — far
   safer than command wildcards; it grants reads in one folder, not arbitrary execution).
   If a full checkout must stay in the scratchpad instead, add that scratchpad root the
   same way.

If you truly cannot bring the repo in-scope, fall back to studying it in the scratchpad —
but expect prompts, and prefer the tool-first convention below to minimize them.

### Code-inspection convention (you and every agent you spawn)

**Read code with the Read / Grep / Glob tools, not with Bash.** The Read tool takes a
line offset+limit for exact ranges (never `awk 'NR>=x'` or `sed -n`), Grep searches
content, Glob finds files. This is faster, doesn't prompt on external paths, and — unlike
per-command `awk`/`grep` invocations — never pollutes the permission allowlist with
one-off entries. Reserve **Bash** for things the tools genuinely can't do: `git log` /
`git blame` / `git shortlog` for origin-story numbers and dates, file counts, and the
`git clone` above. State this convention in the prompt of every agent you spawn.

## Phase 1 — Survey the repo

Goal: a **repo dossier** — the raw material the book plan is built from.

1. Do a quick orientation yourself (top-level `ls`, README, manifest files like
   `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`) to identify the major
   subsystems and the project's rough size and type.
2. Spawn `repo-scout` agents — one per major subsystem/concern, **in parallel, in a
   single message**. Typical cuts for a mid-size repo (adapt to what you saw):
   - purpose, domain, and user-facing behavior (README, docs, examples, CLI/API surface)
   - architecture and control flow (entry points, main loop, module dependency shape)
   - core data structures and domain model
   - external dependencies, frameworks, and the *concepts* they bring in (each one is a
     candidate for "must be taught before use")
   - build / test / run / deploy machinery
   For a small repo (< ~30 files), 1–2 scouts suffice; for a large monorepo, add scouts
   per package. Tell each scout the repo path, its assigned concern, and that its report
   becomes source material for a textbook.
3. Merge the scout reports into `<output-dir>/_notes/dossier.md`. Resolve
   contradictions between scouts by checking the code yourself. The dossier must include
   a **concept inventory**: every non-obvious concept, tool, term, or technique the repo
   relies on, each marked "reader likely knows / must be taught".

## Phase 2 — Book plan (negotiate until approved — the gate)

Draft `<output-dir>/_notes/plan.md` containing:

1. **Audience & prerequisites** — who the reader is, what they're assumed to know.
2. **Language** of the book (English / 中文 / mixed).
3. **Narrative arc** — the one-paragraph story the book tells, from "why does this
   project exist and what problem shaped its design" down to "you could now modify it".
4. **Chapter list** — for each chapter: title, one-paragraph goal, the questions it
   answers, key files/code it walks through, and the concepts it *introduces* vs.
   concepts it *requires* (from earlier chapters).
5. **Concept dependency order** — show that every concept is introduced before any
   chapter uses it. This ordering constraint drives the chapter sequence.
6. **Depth calibration** — which areas get line-level code walkthroughs vs. an
   architectural treatment, and why.
7. **Estimated size** — rough page/word count per chapter, so the user can push back on
   scope.
8. **Book title and output directory name** — propose both (directory name short and
   kebab-case, e.g. `understanding-<repo>/`); the user decides in negotiation. Rename
   the provisional directory once settled.
9. **Signature devices** — a sketch of the master diagram (the whole-system map in the
   big-picture chapter that later chapters zoom into, regions annotated with chapter
   numbers); whether the book ends with a capstone case-study chapter tracing one real
   feature end-to-end; whether chapters include exercises (思考题); and the 2–3 named
   reading routes the README will offer (e.g. ten-minute overview / case-first /
   cover-to-cover). See "Devices that earn their place" in the pedagogy contract.

Then present the plan to the user **in the conversation** (summarize it readably; don't
just say "see the file") and negotiate:

- Use `AskUserQuestion` for the discrete choices (language, audience level, depth,
  whether to include exercises), and open discussion for outline structure.
- Revise `plan.md` after every round of feedback and re-present the changed parts.
- Repeat until the user clearly says the plan is approved (e.g. "看起来不错，开始写吧" /
  "approved" / "go ahead"). Enthusiasm about one aspect is not approval of the whole
  plan. **Do not start Phase 3 without it.** If the user is unavailable mid-negotiation,
  stop the turn after presenting the plan — the plan on disk makes resuming cheap.

## Phase 3 — Write the chapters

Initialize `<output-dir>/_notes/ledger.md`: a running record of, per completed chapter,
(a) concepts introduced and the exact term used for each, (b) notation/metaphor choices,
(c) a 3–5 sentence summary. This is how later chapters stay consistent with earlier ones
and avoid re-explaining or, worse, using an unintroduced term.

Compute writing **waves** from the concept-dependency order in the plan: a chapter may
be written once every chapter it depends on is written. Within a wave, spawn
`chapter-writer` agents in parallel (single message); between waves, update the ledger
from each writer's returned ledger entries before starting the next wave.

Each `chapter-writer` gets: the pedagogy guide path, the dossier path, the plan path,
the current ledger path, its chapter brief (copied inline from the plan), the output
file path (`<output-dir>/NN-slug.md`), and the repo path. Writers read real code and
quote it with `path:line` references — they must not invent code.

After each chapter is written, run a `pedagogy-reviewer` agent on it. The reviewer
returns a findings list (violations of the pedagogy contract). If there are findings,
send the chapter back to a `chapter-writer` in revise mode with the findings. One
review→revise round per chapter by default; escalate to the user only if a chapter fails
review twice.

Pipeline, don't barrier: a chapter can be under review while the next wave's chapters
are being written, as long as ledger ordering is respected.

## Phase 4 — Assemble the book

Spawn a single `book-editor` agent over the whole output directory to:
- write `README.md` (title, how-to-read-this-book guide, table of contents with
  chapter one-liners), a preface, and `glossary.md` compiled from first-occurrence
  definitions;
- enforce terminology consistency and add cross-references ("as we saw in Chapter 3…");
- smooth chapter transitions so the book reads as one narrative, not stapled essays.

Then verify yourself: every chapter file exists, TOC links resolve, no chapter still
contains reviewer TODO markers. Tell the user where the book is, its final structure,
and that `_notes/` can be deleted or kept for future revisions.

## Failure handling

- A scout or writer returning null/garbage: respawn once with a tightened prompt; if it
  fails again, do that piece yourself.
- Mid-run interruption: every phase persists to `_notes/`, so on re-invocation check
  `_notes/` first and resume from the last completed artifact instead of restarting.
