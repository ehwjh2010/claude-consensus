# Consensus Workflow

## Isolation Model

One user requirement maps to exactly one fresh consensus subagent and one new Claude session.

- The main Codex agent reads enough local context to infer the input kind, targets, and starting instructions.
- The main Codex agent creates a fresh subagent for the current requirement.
- The subagent starts a new Claude session on its first review call by omitting `--session`.
- The subagent may reuse the returned Claude `session_id` only inside that same subagent and only for the same requirement.
- A new requirement must create a new subagent and a new Claude session.
- Never reuse an old consensus subagent or old Claude session for a different requirement.
- The main Codex agent must not store, restore, or reuse Claude session ids.
- Claude session ids stay inside the consensus subagent and are not returned for main-agent reuse.
- No `run_id`, manifest, or runtime index is needed. `.runtime/` only stores response markdown files.

## Main Codex Workflow

1. Read enough local context to identify the task, inferred input kind, target files when relevant, and a concise starting plan or file-review instruction for the subagent.
2. Infer the internal input kind:
   - `plan`: the user wants Claude to review a proposal, plan, approach, strategy, or intended edits before work proceeds.
   - `file`: the user wants Claude to review, modify, polish, proofread, or validate existing file or directory contents.
   - When both a plan and files are mentioned, choose the core object Claude needs to judge or help modify.
   - Ask the user only when the object to review cannot be inferred safely.
3. Spawn a fresh subagent without unrelated historical context.
4. Give the subagent only the current requirement context:
   - Original user request.
   - Initial Codex plan or file-review instruction.
   - Inferred input kind and target files if applicable.
   - Workspace path.
   - Relevant constraints and assumptions.
5. Wait for the subagent to complete the Claude review, edits when requested, and Claude rereview loop.
6. Report the subagent result to the user:
   - Complete `Final approved plan`, only for `input-kind=plan` with final status `APPROVED`.
   - Files modified.
   - Main changes made.
   - Number of Claude review rounds.
   - Final status.
   - Any `BLOCKED` reason or explicitly deferred notes.

For `plan`, when the subagent returns final status `APPROVED`, the main Codex agent must first replace its own intended next steps with the subagent's complete `Final approved plan` and treat that plan as the authoritative plan after consensus. The main Codex agent must show the user the complete `Final approved plan`, not just a summary and not the initial plan. If the user asked Codex to proceed after consensus, first show the complete final approved plan, then execute or report only from that final approved plan. For `file`, the target files on disk are the authoritative final state, and the subagent result should summarize modified files, expanded files, deferred notes, verification, round count, and status rather than returning full file contents unless the user explicitly requests them.

The main Codex agent should not call Claude directly for the consensus loop. For `file`, it should not perform the requested file edits after the subagent returns because the subagent owns the review-modify-rereview loop and the workspace edits.

## Caller Constraints

The caller may pass request-specific constraints to this skill, such as read-only references, writable targets, review focus areas, files that must not be changed, or stage-specific risks. Treat these constraints as part of the current consensus run.

This skill still owns the consensus protocol: isolated Claude session, status handling, review / revise / rereview loop, and final reporting. Claude remains read-only through explicit Claude CLI tool restrictions: the wrapper allows only `Read`, `Grep`, `Glob`, and `LS`. Any file changes are performed by the consensus subagent according to Claude's feedback and the caller-provided writable-target constraints.

If caller constraints conflict with this skill's isolation, status, Claude-read-only, or final-reporting rules, this skill's rules take precedence.

## Consensus Subagent Workflow

Use `scripts/ask_claude_consensus.sh` on Unix-like systems or `scripts/ask_claude_consensus.ps1` on PowerShell hosts.

1. Start with the initial plan or file-review instruction and inferred context from the main Codex agent.
2. Round 1: call the script without `--session`, using `--input-kind plan` or `--input-kind file`.
3. For `file`, pass every target file or directory with `--target`.
4. Save the `session_id=...` printed by the script for this requirement only.
5. Read the generated `output_path=...` markdown.
6. Normalize Claude's first meaningful status line before branching:
   - Strip leading whitespace.
   - Strip common markdown wrappers or heading markers such as `**APPROVED**`, `# REVISE`, and `BLOCKED:`.
   - Treat only normalized `APPROVED`, `APPROVED_WITH_NOTES`, `REVISE`, or `BLOCKED` as status tokens; preserve the full markdown body for human review.
7. If the normalized status is `APPROVED`, stop and return by input kind:
   - For `plan`, return `Final approved plan` containing the complete current plan Claude approved, plus final status, deferred notes if any, and Claude review round count.
   - For `file`, return modified files, expanded files, edit summary, verification summary, deferred notes if any, final status, and Claude review round count. Do not return full file contents unless the user explicitly requested them.
8. If the normalized status is `APPROVED_WITH_NOTES`, incorporate the notes into the plan or target files when appropriate, or explicitly defer low-risk notes with a short rationale. Then call the script again with the same `--session` and same `--input-kind`; `APPROVED_WITH_NOTES` is not a terminal status even when no file content changes.
9. If the normalized status is `REVISE`, apply the requested concrete changes to the plan or target files when possible. Then call the script again with the same `--session` and same `--input-kind`.
10. If the normalized status is `BLOCKED`, do not invent missing facts. Stop and return by input kind:
   - For `plan`, return the blocker, `Current best-known plan` containing the complete latest revised plan, deferred notes if any, and review round count.
   - For `file`, return the blocker, current file state summary, modified files if any, expanded files if any, deferred notes if any, and review round count.
11. If no status token is clear, ask Claude to restate the decision with a valid first status line and continue the same round count.
12. Do not impose a fixed round limit. Continue until Claude returns `APPROVED` or `BLOCKED`.

For resumed `plan` rounds, pass the complete updated plan in `--plan`. Do not repeat the original task or summarize what changed.

For resumed `file` rounds, pass the current targets again. Do not repeat the original task, restate what Claude is reviewing, or summarize file edits. Claude uses the same session context plus the current target file contents.

Claude is read-only in this workflow. The wrappers enforce this by passing `--tools "Read,Grep,Glob,LS"` on both new and resumed Claude invocations, so Claude can inspect context but cannot use mutation tools. Claude reviews plans or target file contents, builds the necessary workspace context in the first round, reuses that context in later rounds via the resumed session, and reads additional files only when the revised work introduces new scope, the prior context is missing, or the information may have changed. Claude asks blocking questions, identifies incorrect assumptions, requests specific changes, and calls out missing context or test gaps. Claude must not edit files.

The scripts default new Claude sessions to native `--permission-mode auto`, not `plan`. Native Claude Code plan mode is avoided by default because `ExitPlanMode` is an approval workflow and can interrupt or abort non-interactive consensus runs. The explicit `--permission-mode <mode>` / `-PermissionMode <mode>` override remains available only for new sessions; resumed sessions use `--resume` plus the same read-only tool restrictions.

The consensus subagent may edit workspace files. It should default to editing only the user-specified or inferred target files. If Claude explicitly identifies related files that must change to complete the task correctly, the subagent may expand the write set to those related files. The subagent must list every expanded file in its final result. It must not modify unrelated files.

## Internal Input Kind

`plan` and `file` are internal input kinds. They are not user-facing options and are not review stages.

- `plan`: Use when Claude is reviewing a Codex plan before the main work proceeds. Pass the plan text in `--plan`.
- `file`: Use when the user asks Claude to review one or more existing files or directories. Pass each explicit file or directory path with `--target`; the targets are the source of truth.

For file review requests, do not require the user to phrase the task as a plan review. Create concise instructions that say the subagent will use Claude's review to edit the target if Claude requests changes, then call the script with `--input-kind file --target <path>`.

Verification text is optional supporting context. Use `--verification` or `--verification-file` only when command output or test results are directly relevant.
