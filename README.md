# peer-pr-review

A Claude Code skill that reviews a teammate's GitHub pull request with a Sonnet high-effort subagent. It checks four things:

1. The repo's git workflow policy (branch, title, commits, PR template).
2. The PR's claims against the diff, and whether the change is efficient.
3. Whether the Copilot review ran on the latest commit and its comments were addressed.
4. DRY: duplicated code and re-implemented helpers.

It is read-only. It never posts to GitHub unless you ask.

## Install

```bash
mkdir -p ~/.claude/skills/peer-pr-review
curl -fsSL https://raw.githubusercontent.com/prithivrajmu/peer-pr-review/main/skills/peer-pr-review/SKILL.md \
  -o ~/.claude/skills/peer-pr-review/SKILL.md
```

Needs the `gh` CLI, signed in.

## Use

Say "review PR 123" in Claude Code.
