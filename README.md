# maketutorial

Turn a codebase — or a pile of blog posts and papers — into a **textbook**: a real book,
written in connected prose, that teaches its subject from big-picture design philosophy all
the way down to the details. Not a summary, not API docs, not a cheatsheet.

It's a [Claude Code](https://claude.com/claude-code) **skill**: you run it inside Claude
Code with `/make-textbook`, answer a few questions about scope, approve a plan, and a team
of AI agents researches, writes, reviews, and assembles the book chapter by chapter.

Two kinds of input are supported:

- **A code repository** (local path or GitHub URL) → a book that teaches that codebase,
  design rationale down to line-level implementation.
- **One or more content URLs** (blog posts, papers, articles, docs) → a book that teaches
  the *topic* those pages cover, grounded in them plus its own cited web research.

## See an example

[`understanding-evoscientist/`](understanding-evoscientist/) in this repo is a complete
book this skill produced from a codebase: 17 chapters plus a preface, glossary, and a
"how to read this book" README. Skim it to see what the output looks like before you run
your own.

## Requirements

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** installed and working
  (the CLI, or the desktop / IDE integrations).
- **[GitHub CLI](https://cli.github.com/) (`gh`), authenticated** — run `gh auth login`
  once. This is used to create/clone the output repo and push finished books. (If you skip
  this, everything still works locally; you just publish by hand.)
- **git**, and **web access** (needed for sources mode's research and for cloning remote
  repos).

## Getting started

```bash
git clone https://github.com/LI-SUJU/maketutorial.git
cd maketutorial
claude            # start Claude Code in this folder
```

Then, inside Claude Code:

```
# Not sure what to do? Just run it with no arguments and it explains your options:
/make-textbook

# Teach a codebase (repo mode)
/make-textbook /path/to/some-repo
/make-textbook https://github.com/org/repo

# Teach a topic from articles/papers (sources mode) — pass 1..n URLs, any mix
/make-textbook https://a-blog.com/post https://arxiv.org/abs/1234.5678
/make-textbook https://paper1 https://paper2 https://blog3

# `/mt` is a short alias — identical behavior
/mt https://paper1 https://paper2
```

You don't have to memorize the forms: run `/make-textbook` with nothing and it prints a
numbered menu of what you can do and exactly how to reply.

## What happens when you run it

1. **It confirms what you meant** before doing any work — "a textbook about *this* repo /
   *these* sources, right?" — so a misread never wastes effort.
2. **Sources mode only — topic check.** Your URLs are read and clustered. If they all share
   one theme, that becomes the book's spine. If they split into unrelated topics, you're
   shown a table of *which URL belongs to which topic* and asked which one to build — it
   won't silently mash unrelated things into one book.
3. **You shape the book.** During planning you choose its **size** (Primer → Comprehensive)
   and **structure** (progressive-depth, thematic parts, problem-driven, or
   source/component-anchored), the language, the audience, and the depth.
4. **You approve the plan.** Nothing is written until you say go — push back over as many
   rounds as you like. The plan (and all research notes) are saved to disk, so you can stop
   and resume later.
5. **It writes, reviews, and assembles** the whole book, then publishes it (see below).

### Where books go

Every book is published to **your own GitHub repo named `ai_generated_textbooks`** — one
repo that collects every book you generate, each as its own top-level directory. On the
first run, if that repo doesn't exist yet, you're asked before it's created (private by
default). It's cloned locally as a **sibling of this project** (`../ai_generated_textbooks/`,
its own separate repo), and after assembly the book is committed and — once you confirm the
first push — pushed to your GitHub. Pass an `output-dir` argument to send a one-off book
somewhere else instead.

## The workflow under the hood

Both modes converge on the same **plan → write → assemble** pipeline; they differ only in
how the subject is first surveyed. Each stage is handled by a dedicated agent (see
[Layout](#layout)):

- **Survey.** *Repo mode:* parallel `repo-scout` agents map the codebase into a dossier
  (architecture, domain model, dependencies, concept inventory). *Sources mode:*
  `source-ingest` agents fetch and classify each URL, then `researcher` agents fan out on
  the chosen theme, **adversarially verifying** each claim against a second source, and
  everything lands in a cited dossier with a bibliography.
- **Plan.** A book plan (audience, arc, chapters, concept-dependency order, depth, size,
  structure) is drafted and negotiated with you — the hard gate before any writing.
- **Write.** `chapter-writer` agents write chapters in dependency order (a concept is
  always taught before it's used); each chapter is checked by an adversarial
  `pedagogy-reviewer` and revised until it passes. Everything is grounded in real evidence
  — repo mode quotes code with `path:line`; sources mode cites `[S#]`/`[R#]` keyed to the
  bibliography — never invented.
- **Assemble.** A `book-editor` unifies terminology, adds cross-references, and builds the
  preface, table of contents, glossary, and (sources mode) a `references.md` bibliography.

## Writing principles

The pedagogy every agent follows lives in
[`.claude/skills/make-textbook/references/pedagogy.md`](.claude/skills/make-textbook/references/pedagogy.md)
and is enforced by the reviewer agent:

- 先高层后细节 — big picture and design rationale before mechanism.
- 深入浅出，粗中有细 — every topic in three layers: intuition → mechanism → real evidence.
- 先解释后运用 — a concept/tool is taught from zero before it's used.
- 生僻词首次出现即解释 — every rare term defined at first occurrence (and collected into a
  glossary).
- 教科书而非速查表 — connected prose with openings, closings, and transitions, not
  bullet-point fact dumps.

## Customization

- **Size & structure** are chosen interactively each run — from a short primer to a
  comprehensive treatment, in whichever structural shape fits the material.
- **Faithful vs. researched (sources mode).** By default the book expands beyond your URLs
  with cited web research to cover prerequisites and gaps. Ask to *stay faithful to the
  supplied URLs only* for a tighter survey of exactly what you provided.
- **Install it everywhere.** To use the skill in any project (not just this folder), copy
  `.claude/skills/make-textbook/` and `.claude/skills/mt/` into `~/.claude/skills/`, and the
  six agent files from `.claude/agents/` into `~/.claude/agents/`.

## Layout

```
.claude/
  skills/
    make-textbook/
      SKILL.md               # the orchestration workflow (both modes)
      references/pedagogy.md # the writing contract all agents follow
    mt/SKILL.md              # short alias for /make-textbook
  agents/
    repo-scout.md            # repo mode: read-only repo analyst (parallel)
    source-ingest.md         # sources mode: fetch + classify + cache one URL (parallel)
    researcher.md            # sources mode: verified web research on one sub-question
    chapter-writer.md        # writes/revises one chapter (either grounding mode)
    pedagogy-reviewer.md     # adversarial check against the writing contract
    book-editor.md           # final whole-book consistency pass
understanding-evoscientist/  # an example book produced by this skill
```

Each generated book keeps its working artifacts (`_notes/`, and `_sources/` in sources
mode) alongside the chapters — keep them to make future revisions cheap, or delete them.
