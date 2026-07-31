# maketextbook — project guide for Claude

This project's whole purpose is the **make-textbook** workflow: turn a code repository, or a
set of blog/paper/article URLs, into a real **textbook** (connected prose that teaches a
subject from design philosophy down to detail — not a summary or cheatsheet).

## When the user asks for a book, run the skill

If the user types **`make-textbook`**, **`mt`**, or asks in their own words to "生成教程 /
教科书 / a textbook / a deep-dive book" for a repo or some URLs — **invoke the
`make-textbook` skill** (via the Skill tool) and follow it exactly. Do not improvise a
different flow; `SKILL.md` is the single source of truth. `mt` is the short alias.

The skill's contract for how the book is *written* lives in
`.claude/skills/make-textbook/references/pedagogy.md`; every writing/review agent follows it.

## Two things the user cares about beyond a first draft

- **Detail level (详略) + deepening later.** A book can be written **精简版 (concise)** first —
  a fast, readable pass — and later **deepened into a detailed 版本** without redoing the
  research. The `_notes/` (dossier, plan, ledger) and cached `_sources/` are kept precisely
  so "出一个更详细的版本" reuses them (SKILL.md, *Phase E — Deepen an existing book*). When a
  user says a book is too thin or asks for more depth, reach for that mode, not a fresh run.
- **Clickable citations.** External sources (papers, blog posts, web pages) are cited as
  **clickable links** — inline `[[S3]](url)` and a linked References section — so a reader
  can click through. If a source has a URL, the link is present.
- **Figures.** Useful figures are pulled into the book: we draw our own (Mermaid/ASCII/
  tables) where a concept draws cleanly, and **download** genuinely useful real images into
  the book's `assets/figures/`, embedding them locally with a source line. The `figure-ingest`
  agent fetches and *looks at* each image (Read has vision) before captioning it. Policy is
  simple — useful → download → attribute source; no licensing gate (books stay in the user's
  private repo). A figure locked in a PDF, or better drawn fresh, is redrawn as Mermaid or
  linked in prose instead.

## Allowed tools / permissions (so the run stays quiet)

The workflow only needs a small, stable set of shell commands and directory access. These
are **pre-approved in `.claude/settings.json`** (committed) so they don't prompt — this file
just documents intent; `settings.json` is what actually grants them. If you hit a prompt for
something on this list, the grant is missing — add the specific form to `settings.json`
`permissions.allow` rather than working around it.

- **Reading code/sources:** use the **Read / Grep / Glob** tools (not Bash `cat`/`sed`/`awk`)
  and **WebFetch / WebSearch** for the web. These don't prompt on external paths.
- **git:** `clone`, `fetch`, `add`, `commit`, `push`, `pull`, `mv`, `-C`, `log`, `blame`,
  `shortlog`, `checkout`, `branch`, `merge`, `remote`, `worktree` (read-only forms).
- **gh:** `gh auth`, `gh repo`, `gh pr`, `gh api` — create/clone the output repo, push books.
- **curl:** `curl -L` — only the `figure-ingest` agent uses it, to fetch page HTML and
  download useful figure images into the book's `assets/figures/`.
- **Writing books:** every book lands in the sibling repo
  `../ai_generated_textbooks/`, registered in `permissions.additionalDirectories` so writes
  there are prompt-free. Never write generated books into this project.
- **WebFetch domains** are allow-listed per book as they come up (in `settings.local.json`);
  a first fetch of a new domain may prompt once — that's expected and safe.

Machine- and book-specific grants (specific WebFetch domains, `pip`, a local study-repo
path) live in the un-committed `.claude/settings.local.json`.
