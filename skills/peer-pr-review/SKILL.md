---
name: peer-pr-review
description: Review a GitHub pull request for policy compliance, claim accuracy, Copilot review status and DRY, in a Sonnet 5.5 high-effort subagent. Two modes. Peer mode (a teammate's PR) reports in the terminal and drafts a message to the author. Self mode (your own PR) requests a Copilot review if missing, waits for it, and posts the findings as one PR comment. Use whenever the user says review this PR, check PR 123, "can you review <PR URL>", self review, review my PR, or after you open a PR in a repo whose instructions say to self-review, even if they never say "skill".
---

# Review a PR

Four checks, one verdict. The review runs in a subagent so the diff and file reads stay out of the main context.

## Step 1: resolve the PR and the mode

Accept a number, a URL, or nothing (then the current branch's PR). The main session fetches only what it needs to choose the mode; the subagent fetches the rest:

```bash
gh pr view <PR> --json number,url,author,headRefOid
gh api user --jq .login
```

**Self mode** when the author is the current user or the user says "self review"; otherwise **peer mode**.

Never check out the PR branch in a checkout other sessions may share. To read code beyond the diff use `git fetch origin pull/<n>/head:refs/pr/<n>` and `git show refs/pr/<n>:<path>`.

## Step 2 (self mode only): get a Copilot review on the head commit

Copilot's login is `copilot-pull-request-reviewer[bot]` (sometimes `Copilot`); match on the prefix `copilot`, ignoring case.

1. If a Copilot review exists with `commit_id` equal to `headRefOid`, go to Step 3.
2. Else request it: `gh pr edit <n> --add-reviewer @copilot` (safe to repeat; `requested_reviewers` does not list Copilot, so do not check it first). If that fails (Copilot review not enabled for the repo, old `gh`), say so in the comment and go to Step 3.
3. Wait in the background (`run_in_background`, no foreground sleep), up to 15 minutes:

```bash
for i in $(seq 30); do
  gh api repos/{owner}/{repo}/pulls/<n>/reviews --paginate --jq '.[] | select(.user.login|ascii_downcase|startswith("copilot")) | .commit_id' | grep -q <headRefOid> && exit 0
  sleep 30
done; exit 1
```

On timeout, continue and say in the comment that the Copilot review had not arrived.

## Step 3: launch the reviewer

Call the Agent tool with `model: "sonnet"` and `effort: "high"`. If the call is refused, tell the user once and run the review inline.

Give the subagent the PR number, the repo path, the absolute path of this file, and the instruction to run "The four checks" on that PR. It reads only; it posts nothing. It fetches:

```bash
gh pr view <PR> --json number,title,body,author,baseRefName,headRefName,headRefOid,commits,files,reviews,statusCheckRollup,url
gh pr diff <PR>
```

## The four checks

### 1. Git workflow policy
Find the repo's policy: `docs/GIT_WORKFLOW_POLICY.md`, else `CONTRIBUTING.md`, else search `docs/` for "workflow" or "branch". Read it and CLAUDE.md's git and deployment sections. If none exists, say "no policy found" and skip. Otherwise check the PR against what the document actually says, typically:
- branch name format and length limits
- PR title and commit message format (type prefix, ticket key)
- atomic, reviewable commits; no leftover fixup or "wip" commits; tests in the same commit as the change
- PR template sections filled in; screenshots for UI changes (one per reachable state)
- no committed files the policy forbids; no attribution trailers if the policy forbids them

Quote the rule next to each violation so the author can verify it.

### 2. Claims versus code
List every concrete claim in the PR title, body, commit messages and code comments ("fixes X", "no behavior change", "N tests added", "faster", "covers all brands", "migration is safe"). For each, find the evidence in the diff and mark **Verified**, **Not supported**, or **Cannot verify** (say what would verify it). Then judge efficiency: does the change do what it claims in the simplest way? Flag extra queries or requests, work in a loop, a new abstraction used once, a new dependency for a few lines, or a heavier approach where an existing helper fits. Check that tests assert the claimed behavior rather than just run. For a deep audit with fixes, point the user to `/verify-claims` rather than growing this check.

### 3. Copilot review and CI
```bash
gh api repos/{owner}/{repo}/pulls/<PR>/reviews --paginate
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100){nodes{isResolved isOutdated path line comments(first:50){nodes{author{login} body}}}}}}}' -f o=<owner> -f r=<repo> -F n=<PR>
```
Report:
- **Done?** Did a Copilot review run on the head commit? If commits landed after it, the review is stale.
- **Each Copilot comment:** fixed in a later commit (check the code at that path), replied to with a reason, or unaddressed. Resolved without a fix or a reply counts as unaddressed; do not trust "resolved" alone.
- **CI:** one line from `statusCheckRollup`: passing, failing (name the check), or pending.

### 4. DRY
Look for duplication in the diff against itself and against the existing codebase. Grep for existing helpers, types, constants and components that do the same job. If the repo's CLAUDE.md or architecture doc names shared helpers and "always use X" rules, read it first and treat a bypass of a named helper as a finding. Flag:
- copy-pasted blocks or near-identical functions, components, queries or tests
- re-implemented utilities that already exist
- literals that repeat and belong in a constant or a shared source (labels, column names, ids)
Only report real repetition with file:line references on both sides. Three similar lines are not a violation; do not push for abstraction that has no second use.

## Several PRs at once

Launch one subagent per PR in a single turn, each given this file's absolute path and its one PR. Wait for all of them, then consolidate. Do not summarize a PR you have no report for.

## Output: peer mode

**Part 1: report in the terminal.** A table of PRs (number, author, verdict, main finding), then per PR the verdict (Approve / Approve with nits / Request changes, one sentence why) and one short section per check, each finding as `file:line` plus the rule or evidence. Say which findings you verified and which you took from a subagent.

**Part 2: one message per author.** Group the PRs by author; one message per author covering all their PRs.
- Plain text in a single code block per author, no markdown styling, ready to paste. Put the author's login above the block, outside it.
- Open with the author's first name (or login if unclear). Address them as "you"; do not guess pronouns.
- One or two lines on what is good and the overall state (CI, Copilot, whether it can merge), then one block per PR headed `#<n> <title>`.
- Inside each PR, a numbered list of fixes, must-fix-before-merge first, then `Optional / follow-up:` for the rest. Each item: what, where (`file:line`), why in one sentence, and the suggested fix. Leave out anything you cannot point to.
- Close with merge-order advice when PRs are stacked.

Never post a review, comment, approval or resolve a thread in peer mode unless the user asks after seeing the report; a posted review is visible to the whole team and cannot be unsent.

## Output: self mode

Post one PR comment, and re-runs edit it rather than adding another. The body starts with the marker line `<!-- self-pr-review -->`:

```bash
me=$(gh api user --jq .login)
id=$(gh api repos/{owner}/{repo}/issues/<n>/comments --paginate --jq ".[] | select(.user.login==\"$me\" and (.body|startswith(\"<!-- self-pr-review -->\"))) | .id" | head -1)
if [ -n "$id" ]; then gh api -X PATCH repos/{owner}/{repo}/issues/comments/$id -F body=@body.md
else gh pr comment <n> --body-file body.md; fi
```

Body: `## Self review (<short head SHA>)`, the verdict in one sentence, then one section per check with findings as `file:line` plus the rule or evidence, must-fix first, and "No issues found" for a clean check. In the Copilot section, list each Copilot comment and its state. Write the file in the scratchpad, not the repo.

Self mode only reports. Do not approve, resolve threads, reply to Copilot, or push fixes. Afterwards, give the user the comment URL and the must-fix items in the terminal.

Both modes: be calibrated. Separate what you verified from what you inferred, and say "no issues found" for a clean check rather than inventing nits.
