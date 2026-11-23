---
name: peer-pr-review
description: Review a teammate's GitHub pull request for policy compliance, claim accuracy, Copilot review status and DRY. Runs the review in a Sonnet 5.5 high-effort subagent. Use whenever the user says review this PR, look at my teammate's PR, check PR #123, "can you review <PR URL>", or asks whether a PR follows the git workflow, whether its claims hold, whether Copilot comments were handled, or whether it repeats code, even if they never say "skill".
---

# Review a teammate's PR

Four checks, one verdict. The review runs in a subagent so the diff and file reads stay out of the main context.

## Step 1: resolve the PR

Accept a number, a URL, or nothing (then use the current branch's PR: `gh pr view --json number`). Fetch metadata once:

```bash
gh pr view <PR> --json number,title,body,author,baseRefName,headRefName,commits,files,reviews,statusCheckRollup,url
gh pr diff <PR>
```

Do not check out the branch in the main checkout (other sessions share it). If code beyond the diff must be read, use `gh api` / `git show origin/<headRef>:<path>`, or a worktree under `.claude/worktrees/`.

## Step 2: launch the reviewer

Call the Agent tool with `model: "sonnet"` and `effort: "high"` (the user asked for Sonnet 5.5 at high effort; those are the values the tool accepts). If the call is refused, tell the user once and run the review inline.

Give the subagent the PR number, the repo path, and the four checks below. It only reads; it posts nothing.

## The four checks

### 1. Git workflow policy
Find the repo's policy: `docs/GIT_WORKFLOW_POLICY.md`, else `CONTRIBUTING.md`, else search `docs/` for "workflow" or "branch". Read it and CLAUDE.md's git and deployment sections. If none exists, say "no policy found" and skip. Otherwise check the PR against what the document actually says, typically:
- branch name format and length limits
- PR title and commit message format (type prefix, ticket key)
- atomic, reviewable commits; no leftover fixup or "wip" commits; tests in the same commit as the change
- PR template sections filled in; screenshots for UI changes (one per reachable state)
- no direct commits to the default branch; no committed files the policy forbids
- no attribution trailers if the policy forbids them

Quote the rule next to each violation so the author can verify it.

### 2. Claims versus code
List every concrete claim in the PR title, body, commit messages and code comments ("fixes X", "no behavior change", "N tests added", "faster", "covers all brands", "migration is safe"). For each, find the evidence in the diff and mark **Verified**, **Not supported**, or **Cannot verify** (say what would verify it). Then judge efficiency: does the change do what it claims in the simplest way? Flag extra queries or requests, work in a loop, a new abstraction used once, a new dependency for a few lines, or a heavier approach where an existing helper fits. Check that tests assert the claimed behavior rather than just run.

### 3. Copilot review
```bash
gh api repos/{owner}/{repo}/pulls/<PR>/reviews --paginate
gh api repos/{owner}/{repo}/pulls/<PR>/comments --paginate
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100){nodes{isResolved isOutdated path comments(first:5){nodes{author{login} body}}}}}}}' -f o=<owner> -f r=<repo> -F n=<PR>
```
Copilot's login is `copilot-pull-request-reviewer` (or `Copilot`). Report:
- **Done?** Did a Copilot review run, and on the latest commit (compare review `commit_id` to the head SHA)? If commits landed after it, say the review is stale.
- **Each Copilot comment:** resolved, or replied to, or fixed in a later commit (check the diff at that path), or ignored. Resolved without a fix or a reply explaining why counts as unaddressed.
- Do not trust "resolved" alone; confirm the code changed when the comment asked for a change.

### 4. DRY
Look for duplication in the diff against itself and against the existing codebase. Grep for existing helpers, types, constants and components that do the same job. If the repo has a CLAUDE.md or architecture doc that names its shared helpers and "always use X" rules, read it first and treat a bypass of a named helper as a finding. Flag:
- copy-pasted blocks or near-identical functions, components, queries or tests
- re-implemented utilities that already exist
- literals that repeat and belong in a constant or a shared source (labels, column names, ids)
Only report real repetition with file:line references on both sides. Three similar lines are not a violation; do not push for abstraction that has no second use.

## Several PRs at once

Launch one subagent per PR in a single turn (they are independent), each told to read this file and run the four checks on its PR only. Wait for all of them, then consolidate as below. Do not summarize a PR you have no report for.

## Output

Two parts, in this order.

**Part 1: report in the terminal.** A table of PRs (number, author, verdict, main finding), then per PR the verdict (Approve / Approve with nits / Request changes, one sentence why) and one short section per check, each finding as `file:line` plus the rule or evidence. Say which findings you verified and which you took from a subagent.

**Part 2: one message per author.** Read each PR's author from the metadata and group the PRs by author. Write one message per author covering all of their PRs, so a person with three PRs gets one message, not three. Format:

- Plain text in a single code block per author, no markdown styling, ready to paste into chat or a PR comment. Put the author's login above the block, outside it.
- Open with the author's first name (or login if unclear). Do not guess pronouns; address them as "you".
- One or two lines on what is good and the overall state (CI, Copilot, whether it can merge), then one block per PR headed `#<n> <title>`.
- Inside each PR, a numbered list of things to fix, with the changes that should happen before merge first, then a line `Optional / follow-up:` for the rest. Each item says what, where (`file:line`) and why in one sentence, and suggests the fix. Leave out anything you cannot point to.
- Close with ordering advice when PRs are stacked (merge order, rebase after the base merges).
- Short, plain sentences. Kind and specific; no praise that is not earned.

The user's own PRs, if included, get a report but no message.

Never post a review, comment, approval or resolve a thread on GitHub unless the user asks after seeing the report; a posted review is visible to the whole team and cannot be unsent.

Be calibrated: separate what you verified from what you inferred, and say "no issues found" for a clean check rather than inventing nits.
