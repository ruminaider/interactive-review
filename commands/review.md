---
description: "Interactive PR review with parallel agent sweep, self-verification, and triage loop"
argument-hint: "[PR-number-or-URL] [using <reviewer-name>]"
allowed-tools: ["Bash", "Read", "Grep", "Glob", "Task", "AskUserQuestion", "ToolSearch"]
---

# Interactive Review Command

You are running the interactive-review command. This implements a structured PR review workflow with three stages:
1. **Automated Sweep** — parallel review agents compile findings
2. **Self-Verification** — you re-read actual code to confirm/retract each finding
3. **Interactive Triage** — walk through each finding with the user for draft-preview-post decisions

## Phase 1: Determine Review Mode

Parse `$ARGUMENTS` to determine the review mode and reviewer preferences:

### Reviewer Preferences
Before determining mode, scan `$ARGUMENTS` for reviewer specifications. Look for patterns like:
- "using the <name> reviewer" or "with the <name> reviewer"
- "using <name>" where name matches a known agent pattern (e.g., "ghostmonk", "kieran", "security-sentinel")
- Any agent name mentioned explicitly by the user

Extract these as `requested_reviewers` — a list of reviewer names/patterns the user wants included. These will be forwarded to the review-compiler so it can skip interactive selection and use the specified reviewers directly.

### PR Mode
If `$ARGUMENTS` contains a number (e.g., `3780`) or a GitHub PR URL:
1. Extract the PR number
2. Verify the PR exists: `gh pr view <number> --json number,title,state`
3. If the PR doesn't exist or is closed, inform the user and exit
4. Store the PR number, owner, and repo for later comment posting

### Local Mode
If `$ARGUMENTS` is empty, "local", or doesn't match a PR pattern:
1. Verify the branch has changes: `git diff main...HEAD --stat`
2. If no changes, report "No changes to review" and exit
3. Note: comment posting will be disabled in local mode

Announce which mode was detected before proceeding.

## Phase 2: Automated Sweep

Launch the `interactive-review:review-compiler` agent via the Task tool.

Pass it the following prompt based on the mode:

### PR Mode Prompt
```
Review PR #<number> in the current repository.
- Mode: PR
- PR number: <number>
- Requested reviewers: <comma-separated list, or "none" if no preference>
Gather the diff, discover available review agents, dispatch them in parallel, and compile deduplicated findings.
```

### Local Mode Prompt
```
Review the current branch changes against main.
- Mode: Local
- Requested reviewers: <comma-separated list, or "none" if no preference>
Gather the diff using git diff main...HEAD, discover available review agents, dispatch them in parallel, and compile deduplicated findings.
```

When `requested_reviewers` is populated, include them in the prompt so the review-compiler knows to use those specific reviewers instead of asking interactively.

Wait for the review-compiler to return structured findings.

### Error Handling
If the review-compiler fails:
1. Report the error to the user
2. Ask if they want to retry or abort
3. If retry, re-launch the agent
4. If abort, exit gracefully

### Zero Findings
If the review-compiler returns zero findings:
1. Report: "Clean review — no issues found across N agents."
2. List which agents were used
3. Exit

## Phase 3: Interactive Triage Loop

Process each finding in the order returned by review-compiler (severity-sorted: critical first).

### For EACH Finding, Perform These Steps in Order:

#### Step 1: Independent Verification via Subagent (MANDATORY — DO NOT SKIP)

This is the most important step. Each finding MUST be independently verified by a fresh subagent before being presented to the user. Do NOT verify findings yourself — delegate to a subagent to avoid confirmation bias.

**Parallel launch (start of Phase 3):**
At the beginning of the triage loop, launch ALL verification subagents in parallel using multiple Task tool calls in a single message. Use `subagent_type: "general-purpose"` with `model: "haiku"` for each. Run them in the background (`run_in_background: true`) so they execute concurrently.

Each verification subagent receives this prompt:
```
Verify this code review finding against the CURRENT state of the source code.

FINDING:
- File: <file_path>
- Lines: <line_range>
- Summary: <summary>
- Detail: <detail>
- Category: <category>

INSTRUCTIONS:
1. Read the file at the specified lines (include ~10 lines of surrounding context)
2. For type/contract/API claims: Grep for the relevant definition and verify the claim
3. For visual/design concerns: note "Unable to verify against design" unless Figma tools are available
4. Determine if the finding is accurate as of the CURRENT code

RESPOND WITH EXACTLY ONE OF:
- CONFIRMED: <explanation of why the finding is accurate>
- REVISED: <what needs correction about the finding, plus updated details>
- RETRACTED: <why this is a false positive — what the code actually does>
```

**Sequential consumption (during triage):**
Before presenting each finding, read the corresponding verification subagent's output. Use its verdict (CONFIRMED/REVISED/RETRACTED) and explanation in the finding presentation. If a verification subagent hasn't finished yet, wait for it.

**If a verification subagent fails:** Fall back to reading the source code yourself for that finding only, and note "Verification subagent failed — verified manually" in the presentation.

#### Step 2: Present to User

Display the finding in this format:

```
---
### Finding <N>/<total> — <SEVERITY>
**File:** `<file_path>:<line_range>`
**Category:** <category>
**Agents:** <comma-separated agent names>

**Summary:** <one-line summary>

**Detail:**
<full explanation>

**Independent Verification:** <CONFIRMED|REVISED|RETRACTED>
<subagent's verification explanation — what it found when reading the current code>
<for RETRACTED: the subagent's explanation of why this is a false positive>

**Proposed Comment:**
> <soft-toned review comment text — only for CONFIRMED/REVISED>

**Recommendation:** <Comment|Skip>
---
```

For RETRACTED findings:
- Show the retraction explanation prominently
- Recommend "Skip"
- Still let the user override if they disagree with the retraction

#### Step 3: User Decision

Use AskUserQuestion to get the user's decision. Options:

For PR mode:
- **Post** — Post the proposed comment to the PR
- **Edit** — Let the user revise the comment text, then post
- **Skip** — Move to the next finding
- **Skip All Remaining** — End triage early, proceed to summary

For Local mode (no posting):
- **Acknowledge** — Mark as noted, move to next
- **Skip** — Move to next finding
- **Skip All Remaining** — End triage early, proceed to summary

#### Step 4: Execute Decision

**Post (PR mode only):**
Use the GitHub API to post an inline review comment:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  -f body="<comment text>" \
  -f commit_id="$(gh pr view <number> --json headRefOid -q .headRefOid)" \
  -f path="<file_path>" \
  -F line=<end_line> \
  -f side="RIGHT"
```

Confirm the comment was posted successfully. If posting fails, report the error and offer to retry or skip.

**Edit (PR mode only):**
Show the current proposed comment text. The user will provide revised text (they can type it as a response to AskUserQuestion using the "Other" option). Then post the revised text using the same API call above.

**Skip / Acknowledge:**
Record that this finding was skipped and move to the next.

**Skip All Remaining:**
Break out of the loop and proceed to the summary phase.

### Tracking
Maintain running counts throughout the loop:
- `posted`: Number of comments posted to the PR
- `edited_and_posted`: Number of comments edited then posted
- `skipped`: Number of findings skipped by user choice
- `retracted`: Number of findings retracted as false positives
- `acknowledged`: Number of findings acknowledged (local mode)

## Phase 4: Summary Report

After all findings have been processed (or "Skip All Remaining" was selected), present a summary:

```
---
## Review Summary

**Mode:** <PR #N | Local>
**Agents Used:** <N agents>
**Total Findings:** <N>

| Action        | Count |
|---------------|-------|
| Posted        | <N>   |
| Edited+Posted | <N>   |
| Skipped       | <N>   |
| Retracted     | <N>   |

### Posted Comments
<for each posted comment:>
- `<file>:<line>` — <one-line summary>

### Retracted (False Positives)
<for each retracted finding:>
- `<file>:<line>` — <summary> — Reason: <why it was false positive>

### Skipped
<for each skipped finding:>
- `<file>:<line>` — <summary>
---
```

If any agents failed during the sweep, mention them at the bottom:
```
### Agent Failures
- <agent_name>: <error reason>
```

## Tone Guidelines

All proposed review comments MUST follow these tone rules:

1. **Soft and questioning**, not declarative:
   - GOOD: "I think this might cause an issue when..."
   - BAD: "This is wrong because..."

2. **Prefix nits clearly:**
   - "nit: Consider renaming this for clarity"

3. **Suggest, don't demand:**
   - GOOD: "Would it make sense to add a null check here?"
   - BAD: "Add a null check here."

4. **Self-contained:** Each comment should make sense to someone reading it on the PR without additional context. Include enough detail about what the issue is and why it matters.

5. **Acknowledge uncertainty:**
   - "I may be missing context, but it looks like..."
   - "Unless there's a reason I'm not seeing..."

## Behavioral Rules (NON-NEGOTIABLE)

These rules are hard constraints. Violating any of them is a bug in the review process.

1. **NEVER post a comment without showing the draft to the user first.** Every comment goes through the present → decide → execute flow. No exceptions.

2. **ALWAYS verify via subagent before presenting a finding.** Launch a verification subagent for every single finding. Do NOT verify findings yourself — you are biased by having received the finding. No shortcuts, no batching, no "I already read it."

3. **Explicitly call out false positives.** When verification reveals a finding is wrong, say so clearly with the subagent's explanation. This builds trust in the review process.

4. **For local diffs, skip the comment-posting flow entirely.** No `gh api` calls, no "Post" option. Local mode is review-only.

5. **If zero findings, report a clean review and exit.** Don't invent issues to justify the review.

6. **If review-compiler fails, report the error and offer retry.** Don't silently proceed with zero findings.

7. **Respect "Skip All Remaining."** When the user says stop, stop. Don't continue processing findings.
