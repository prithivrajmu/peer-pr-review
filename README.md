# peer-pr-review

A Claude Code skill that reviews a GitHub pull request with a Sonnet high-effort subagent. It checks four things:

1. The repo's git workflow policy (branch, title, commits, PR template).
2. The PR's claims against the diff, and whether the change is efficient.
3. Whether the Copilot review ran on the latest commit and its comments were addressed.
4. DRY: duplicated code and re-implemented helpers.

Two modes:

- **Peer mode** (someone else's PR): read-only. It reports in the terminal and drafts a message to the author. It never posts to GitHub unless you ask.
- **Self mode** (your own PR, or "self review"): it requests a Copilot review if the head commit has none, waits up to 15 minutes for it, then posts the findings as one PR comment. Re-runs edit that comment. It never approves, resolves threads or pushes fixes.

## Install

```bash
mkdir -p ~/.claude/skills/peer-pr-review
curl -fsSL https://raw.githubusercontent.com/prithivrajmu/peer-pr-review/main/skills/peer-pr-review/SKILL.md \
  -o ~/.claude/skills/peer-pr-review/SKILL.md
```

Needs the `gh` CLI, signed in.

## Use

Say "review PR 123" or "self review" in Claude Code. To self-review every PR in a repo, add a rule to its CLAUDE.md: "After you open a PR, run the peer-pr-review skill in self mode on it."
