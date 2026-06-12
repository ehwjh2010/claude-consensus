# Script Usage and Verification

## Script Usage

```bash
./scripts/ask_claude_consensus.sh \
  --workspace /path/to/workspace \
  --input-kind plan \
  --task "Original user request" \
  --plan "Current Codex plan" \
  --round 1
```

By default, the scripts do not pass `--model`, so Claude uses the Claude CLI default. Override it with `--model <name>` on Unix-like systems or `-Model <name>` in PowerShell.

For follow-up plan rounds inside the same consensus subagent and same requirement:

```bash
./scripts/ask_claude_consensus.sh \
  --workspace /path/to/workspace \
  --input-kind plan \
  --plan "Complete updated Codex plan" \
  --round 2 \
  --session "$CLAUDE_SESSION_ID"
```

For a file review inferred from a user request:

```bash
./scripts/ask_claude_consensus.sh \
  --workspace /path/to/workspace \
  --input-kind file \
  --target /path/to/file.md \
  --task "Original user request" \
  --plan "Use Claude's review to edit the file if changes are requested." \
  --round 1
```

For follow-up file rounds after the consensus subagent edits targets:

```bash
./scripts/ask_claude_consensus.sh \
  --workspace /path/to/workspace \
  --input-kind file \
  --target /path/to/file.md \
  --round 2 \
  --session "$CLAUDE_SESSION_ID"
```

The script prints:

```text
session_id=<claude-session-id>
output_path=<markdown-output-path>
```

## Verification

Recommended static checks:

```bash
bash -n scripts/ask_claude_consensus.sh
./scripts/ask_claude_consensus.sh --help
./scripts/ask_claude_consensus.sh --input-kind file --task "x" --plan "y"
./scripts/ask_claude_consensus.sh --input-kind nope --task "x" --plan "y"
```

When `claude` and `jq` are installed, run a dummy two-round review and confirm:

- Round 1 creates and prints a `session_id`.
- Round 2 reuses that session with `--session`.
- `.runtime/*.md` is generated.

When `pwsh` is installed, also verify PowerShell help and argument validation:

```powershell
./scripts/ask_claude_consensus.ps1 -Help
```
