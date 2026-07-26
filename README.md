# maketutorial

A Claude Code skill that turns a code repository into a **textbook** — a book that
teaches the project from its big-picture design philosophy down to line-level detail.

## Usage

Start Claude Code in this folder, then:

```
/make-tutorial /path/to/some-repo
/make-tutorial /path/to/some-repo /path/to/output-dir   # override output location
```

By default each book is written to a new subdirectory of **this folder** — the
directory name (and book title) is one of the things settled with you during plan
negotiation, so this folder accumulates one book per repo you run it on.

The workflow:

1. **Survey** — parallel `repo-scout` agents map the repo into a dossier (architecture,
   domain model, dependencies, concept inventory).
2. **Plan** — a book plan is drafted (audience, language, narrative arc, chapters,
   concept-dependency order, depth calibration) and **presented to you for discussion.
   Nothing is written until you approve the plan** — push back as many rounds as you like.
3. **Write** — `chapter-writer` agents write chapters in dependency order (concepts are
   always taught before they're used); each chapter is checked by a `pedagogy-reviewer`
   and revised until it passes.
4. **Assemble** — a `book-editor` unifies terminology, adds cross-references, and builds
   the preface, table of contents, and glossary.

`_notes/` inside each book directory holds the working artifacts (dossier, plan,
ledger) — keep them to make future revisions cheap, or delete.

## Writing principles

Encoded in `.claude/skills/make-tutorial/references/pedagogy.md`, enforced by the
reviewer agent:

- 先高层后细节 — big picture and design rationale before mechanism.
- 深入浅出，粗中有细 — every topic in three layers: intuition → mechanism → real code.
- 先解释后运用 — a concept/tool is taught from zero before showing how the repo uses it.
- 生僻词首次出现即解释 — every rare term defined at first occurrence (and collected
  into the glossary).
- 教科书而非速查表 — connected prose with openings, closings, and transitions; not
  bullet-point fact dumps.
- All code quotes come from the real repo with `path:line` references.

## Layout

```
.claude/
  skills/make-tutorial/
    SKILL.md                 # the orchestration workflow
    references/pedagogy.md   # the writing contract all agents follow
  agents/
    repo-scout.md            # read-only repo analyst (fanned out in parallel)
    chapter-writer.md        # writes/revises one chapter
    pedagogy-reviewer.md     # adversarial check against the writing contract
    book-editor.md           # final whole-book consistency pass
```

To make the skill available in every project (not just this folder), copy
`.claude/skills/make-tutorial/` to `~/.claude/skills/` and the four agent files to
`~/.claude/agents/`.
