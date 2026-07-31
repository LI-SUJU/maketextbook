---
name: mt
description: Short alias for make-textbook. Turn a code repository OR one or more blog/paper/article URLs into a textbook. Use exactly as you would make-textbook. Accepts a repo path or one or more URLs.
argument-hint: "<repo-path | url ...> [output-dir]"
---

# mt — alias for make-textbook

This is a shorthand. Immediately invoke the **make-textbook** skill (via the Skill tool),
passing through the same arguments this alias received verbatim, and then follow
make-textbook's workflow exactly — including its Phase 0 behavior: show the numbered menu if
no repo/URLs were given, or confirm the detected intent before doing any work.

Do not reimplement anything here; make-textbook is the single source of truth.
