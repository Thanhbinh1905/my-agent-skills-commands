---
name: git-master
description: Git operations override that preserves normal git workflow while forbidding automatic Sisyphus attribution footers and co-author trailers.
origin: local
---

# Git Master Override

Use this skill for git operations such as commit, push, rebase, log, blame, and branch management.

## Core Rules

- Prefix every git command with `GIT_MASTER=1`.
- Follow the repository's existing commit message style.
- Never create a commit unless the user explicitly asks for one.
- Never push unless the user explicitly asks for it.
- Never use destructive git operations unless the user explicitly requests them.
- Do not skip hooks unless the user explicitly asks.

## Commit Message and Footer Policy

- Do **not** append any automatic attribution footer.
- Do **not** append `Ultraworked with Sisyphus`.
- Do **not** append any automatic `Co-authored-by` trailer.
- Do **not** append `Co-authored-by: Sisyphus <clio-agent@sisyphuslabs.ai>`.
- Add commit footers or co-author trailers only when the user explicitly requests them.

## Commit Quality

- Keep commits atomic.
- Stage only relevant files.
- Do not include secrets.
- Prefer concise commit messages that match the repo's established style.

## Push Safety

- Use `--force-with-lease` instead of `--force` when a force push is explicitly required.
- Warn before any push that rewrites remote history.
