# .config/

Repository configuration — **excluded from the Claude Project knowledge base**.

This folder holds files that are versioned in Git but should NOT be ingested as
Project knowledge (so the assistant doesn't treat its own system prompt as a
"document" to answer questions from).

- `claude-project-instruction.md` — the assistant's system prompt. Copy its
  contents into the Claude Project's **custom instructions**, not the knowledge base.

## How the exclusion is configured

Exclusion is set in the Claude Project's **GitHub connector settings** (Claude UI),
not by this file. When connecting `HueChuJitsu/qa-training` to the Project, exclude
the `.config/` path from the synced files so nothing in here is indexed as knowledge.
