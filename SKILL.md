---
name: claude-consensus
description: Use only when the user explicitly asks Codex to have Claude review a plan or existing file content, reach consensus with Claude, run Claude consensus, or perform a similar graded Claude review-and-revision loop.
---

# Claude Consensus

Use this skill only for explicit requests to send a Codex plan or existing file content to Claude for independent graded review and consensus. Do not trigger it for ordinary planning, code review, or editing requests unless the user clearly asks for Claude to be involved.

The user does not need to know the internal protocol. Codex infers whether Claude should review a `plan` or `file` input, starts an isolated consensus subagent, and lets that subagent run the Claude review, apply requested edits when appropriate, and request rereview until Claude returns `APPROVED` or `BLOCKED`.

## Load References

Read these references only when needed for the current consensus run:

- Before executing any consensus run, read `references/consensus-workflow.md` for the full isolation model, main-agent workflow, caller constraints, subagent loop, and internal input-kind rules.
- Before calling `scripts/ask_claude_consensus.sh` or `scripts/ask_claude_consensus.ps1`, read `references/script-usage-and-verification.md` for command examples and validation checks.
- Before constructing, changing, or interpreting Claude's review instructions or statuses, read `references/claude-review-contract.md` for the complete review contract.

## Core Rules

One user requirement maps to exactly one fresh consensus subagent and one new Claude session.

- The main Codex agent reads enough local context to infer the input kind, targets, and starting instructions.
- The main Codex agent creates a fresh subagent for the current requirement.
- The subagent starts a new Claude session on its first review call by omitting `--session`.
- The subagent may reuse the returned Claude `session_id` only inside that same subagent and only for the same requirement.
- Never reuse an old consensus subagent or old Claude session for a different requirement.
- The main Codex agent must not store, restore, reuse, or return Claude session ids for later reuse.
- Claude remains read-only through explicit Claude CLI tool restrictions: the wrapper allows only `Read`, `Grep`, `Glob`, and `LS`.
- The consensus subagent owns the review / revise / rereview loop and any workspace edits for `file` input.
- The main Codex agent should not call Claude directly for the consensus loop.
- Continue until Claude returns `APPROVED` or `BLOCKED`; `APPROVED_WITH_NOTES` is not a terminal status.

For `plan`, when the subagent returns final status `APPROVED`, the main Codex agent must first replace its own intended next steps with the subagent's complete `Final approved plan` and treat that plan as the authoritative plan after consensus. The main Codex agent must show the user the complete `Final approved plan`, not just a summary and not the initial plan.

For `file`, the target files on disk are the authoritative final state, and the subagent result should summarize modified files, expanded files, deferred notes, verification, round count, and status rather than returning full file contents unless the user explicitly requests them.
