# CLAUDE.md

Project guidance for AI agents working in this repository.

## What this is

QA training & onboarding documentation for the **Jitsu** last-mile delivery platform.
Documentation only — no application code.

## Conventions

- **English only.** Every file (docs, README, comments, commit messages) must be written in English. Do not add or leave Vietnamese text.
- **Structure:**
  - `docs/` — reference documentation set (the source of truth). Read [docs/jitsu-context.en.md](docs/jitsu-context.en.md) first.
  - `lessons/` — topic-based lessons, one file per lesson named `NN-lesson-name.md` (e.g. `01-domain-and-terminology.md`).
- **Cross-references** between docs must be **unversioned** (write "Training Document", not "Training Document v4") so links don't go stale.
- Docs are converted from source PDF/DOCX. When converting: fix artifacts (mojibake, broken `&`, page markers), fence code/SQL/JSON blocks, and use proper Markdown tables.

## When editing

- Keep domain content in `docs/`; do not duplicate it here or in the README.
- After adding a lesson, update the lesson index in [README.md](README.md) and [lessons/README.md](lessons/README.md).
