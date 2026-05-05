---
name: address-pr-review
description:
  Use whenever the user asks to address, work through, respond to, or "go
  through" GitHub pull request review feedback — phrases like "address the PR
  comments", "respond to review feedback", "go through unresolved threads", or
  anything that points at a PR's review conversation. Iteratively queries
  unresolved threads, presents recommendations, applies fixes, replies, and
  resolves until nothing remains. Requires interactive user review. Infer the PR
  from the current branch if not named explicitly.
---

# Addressing GitHub PR review feedback

Drive a loop that walks unresolved review feedback on a pull request, lets the
user decide what to do about each item, and applies the chosen actions —
committing fixes, replying, and resolving — until nothing actionable remains.

The loop matters more than any single step. Reviewers may comment while you
work, replies may fail to post, or the user may want to defer something. The
fresh query at the top of each iteration is the source of truth, not your
in-memory list.

## Rules

These are safety and correctness invariants. They always apply, regardless of
circumstances.

1. **User approval is required for every thread.** Do not apply changes, post
   replies, or resolve threads without an explicit decision from the user. This
   is the primary safety mechanism — without it, a wrong dismissal posts a
   public reply under the user's identity, and a wrong fix lands a commit they
   didn't sanction. This applies to all content: code, configs, AI-authored
   docs, the PR description, everything.

2. **Interactive mode is required.** If running in an auto-accept or otherwise
   non-interactive mode, stop and tell the user. Do not proceed by making
   decisions yourself. The failure modes (public replies, unsanctioned commits)
   are bad enough that doing nothing is the right answer.

3. **Always reply before resolving.** The reply is the audit trail. A silently
   resolved thread looks like ignored feedback.

4. **Never edit or delete other people's comments.** Reply to threads; don't
   rewrite history.

5. **Never force-push or rewrite commits from earlier iterations.** Replies
   already posted reference those SHAs. Rewriting turns the references into dead
   links.

6. **Never post replies referencing commits not on the remote.** If a push
   fails, stop. Don't post replies with SHAs that will 404 for reviewers.

7. **Never edit user-authored content without explicit approval.** Even if it
   would be more efficient, even if the loop would otherwise be blocked. Ask, or
   surface the situation and wait.

8. **Auto-generation disclosure must be visible.** Every reply must end with a
   visible line identifying the agent and that the reply was auto-generated. An
   HTML comment alone is not enough — many GitHub clients hide them.

9. **Never request re-review without user confirmation.** Re-review pings real
   people. The user decides whether and from whom.

## Authorship of PR documentation

The PR has documentation attached to it: the PR description and possibly in-repo
spec, plan, design, or README files in the diff. Both kinds need to stay
accurate as decisions are made, but the right update process depends on who
wrote them.

This rule governs how an already-approved change is applied — whether to show a
before/after diff first. It does not override the user-approval requirement
(rule 1). Every thread still needs a decision from the user.

- **You authored it** if you generated it earlier in this conversation and the
  user has not rewritten it. After user approval, apply changes directly; just
  mention what changed so the user can object on the next turn.
- **The user authored it** if they wrote it, pasted it, or there is any doubt.
  After user approval, surface a proposed change (diff or before/after) and wait
  for explicit approval before applying.

When in doubt, treat as user-authored.

## Prerequisites

Before starting, confirm:

- `gh` CLI is authenticated with `repo` scope (write access to the PR is needed
  for reply and resolve mutations).
- `git` is configured and the working tree is clean enough to make commits.
- A target PR exists. If the user did not name one, infer it from the current
  branch.

If any of these is missing, stop and surface the problem before doing anything
else.

## Identify the PR and the author

Fetch the PR metadata. Record:

- **PR number** — used in every subsequent query.
- **PR author** (`author.login`) — treat any review-thread comments from this
  login as if they came from the user you are chatting with. Their wording in a
  thread carries the same weight as instructions in chat. If they conceded a
  point in-thread, that concession stands.
- **Owner and repo** — for fork PRs, the base repo owns review threads (replies
  and resolutions go there) but the head repo is where commits push.

## The loop

Repeat until the unresolved-thread query returns empty (or the user asks to
stop):

1. Query unresolved feedback.
2. Investigate and present a take on each, with a recommended action.
3. Get the user's decisions (confirm, adjust, or defer).
4. Apply fixes as atomic commits, then push.
5. Reply to each addressed thread.
6. Resolve each addressed thread.
7. Sync the PR description and any AI-authored in-PR docs.
8. Go back to step 1.

Track which thread IDs the user has explicitly **deferred**. Exclude them from
step 1's results so the loop can terminate.

---

### Step 1 — Query unresolved feedback

Fetch all review threads on the PR. You need, at minimum:

- **Per thread:** node ID (needed for reply/resolve mutations), `isResolved`,
  `isOutdated`, file path, line number (current and original).
- **Per comment:** node ID, author login, body, creation timestamp, URL, and the
  diff hunk for context.

Paginate if needed. Use the literal `isResolved` field — do not invent your own
definition of "resolved" based on conversation content. A thread where the
reviewer wrote "lgtm now" but didn't click resolve is still unresolved.

From the results:

- Keep threads where `isResolved` is false.
- Drop threads in the session's deferred set.
- Drop threads whose only comments are your own auto-generated replies from
  earlier iterations.

If nothing remains, the loop is done. Confirm with the user and move to
termination.

> **GitHub-specific note:** The `gh api graphql` command with the
> `pullRequest.reviewThreads` connection is the most reliable way to get thread
> node IDs and the `isResolved`/`isOutdated` fields together. MCP and REST
> alternatives may not return thread node IDs suitable for the reply and resolve
> mutations.

Not all review feedback lives in threads. Two other sources are common:

- **Review body comments** — the summary a reviewer writes when submitting.
  These often frame the overall concern that inline threads elaborate on.
- **PR conversation comments** — standalone comments on the conversation tab,
  not attached to any review or code line. Reviewers frequently post these as
  follow-ups after submitting a review, when they realize they forgot something
  or want to add context that doesn't fit a specific file.

Check for both. They often carry high-level concerns or requests that don't map
to a single line of code. Surface them alongside threads in step 2 and apply the
same user-approval workflow.

### Step 2 — Investigate and form a take

For each unresolved thread:

- Read **every** comment in chronological order. Later comments routinely walk
  back, clarify, or escalate the original point. Acting on only the first
  comment is a common failure mode.
- Comments from the PR author are instructions from the user you're chatting
  with. Their position in-thread is part of the instruction set.
- Open the file at the relevant path and line on the current head commit. If the
  thread is marked outdated, the targeted code has since changed — verify
  whether the concern still applies before forming an opinion.
- Form a concise take: what is being asked, whether it's valid, what (if
  anything) you would change, and roughly how big the change is.
- Note threads that have effectively resolved themselves in discussion (e.g.,
  the reviewer wrote "nm, found it").

Reviewers typically flag one instance of a problem rather than repeating
themselves everywhere. After forming a take on a thread, check the rest of the
PR diff for the same or closely related issues and include any you find in your
recommendations.

### Step 3 — Present recommendations and get decisions

Present your takes grouped and with a recommended action for each thread:

- **Fix** — apply a code, config, or doc change to address the feedback.
- **Dismiss** — reply with reasoning explaining why no change is needed. The
  thread still gets resolved.
- **Defer** — skip for this session. Add to the deferred set.

**Lead with your recommendations.** Group threads by recommended action and
present them so the user can confirm the whole batch or adjust individual items.
Include each thread's URL so the user can jump to the conversation. For example:

> I'd recommend fixing these:
>
> 1. **Missing null check in token refresh** ([thread](url)) — reviewer is
>    right, `exp` can be null from this IdP. I also found the same missing check
>    at `auth/session.ts:42` and `auth/verify.ts:88` in the diff.
> 2. **Stale comment on retry logic** ([thread](url)) — the comment references
>    the old timeout value.
>
> And dismissing this one: 3. **Suggesting fail-fast instead of retry**
> ([thread](url)) — the retry is intentional for deploy windows; I'd explain the
> reasoning.
>
> Want me to go ahead with that, or would you like to change anything?

Accept natural responses. "Looks good", "go ahead", "yes" all confirm the
recommendations. "Fix all of them", "dismiss 3 too", "defer 2 for now" are
adjustments. Don't insist on a rigid format or per-thread responses.

If the user's decision contradicts your take, defer to the user. They are the PR
author.

### Step 4 — Atomic commits

For threads being fixed:

- Group changes into commits by **logical relatedness**, not by thread count.
  Two threads pointing at the same bug get one commit. One thread touching three
  independent files may warrant separate commits.
- Each commit should build (and ideally pass tests) on its own. Don't stage half
  a fix that depends on a later commit.
- Write commit messages that describe the change, not the thread. Someone
  reading the log later should understand the change without opening the PR. If
  a thread is the source of truth for _why_, reference it in a trailer (e.g.,
  `Addresses: <thread-url>`).
- If a fix changes behavior described in an in-repo doc that's part of the PR,
  the doc update is part of the fix. Apply the authorship rule to decide whether
  to surface a diff first.

Push once after all commits for this iteration are staged. Record which commit
SHAs address which threads — step 5 needs this mapping.

If the push fails, surface the failure and stop. Do not post replies referencing
commits not on the remote (rule 6).

### Step 5 — Reply to every addressed thread

Reply to **both** fixed and dismissed threads. Each thread gets its own reply on
its own thread — don't combine multiple threads into one comment.

For the reply body:

- **Fixed threads:** reference the commit SHA and briefly state what changed.
  Enough that a reviewer doesn't need to open the diff to see if their comment
  was addressed.
- **Dismissed threads:** explain _why_. The reasoning matters; "won't fix" alone
  is unhelpful. If the thread concluded itself in earlier discussion, point at
  that.
- **All replies:** end with a visible auto-generation disclosure (rule 8).

Post replies using the GraphQL `addPullRequestReviewThreadReply` mutation (it
takes the thread node ID from step 1).

If a reply fails to post, do not resolve that thread. Flag the failure to the
user.

> **GitHub-specific note:** Write commit SHAs and URLs as bare text in reply
> bodies, not inside backticks. GitHub's autolinker only works on unformatted
> SHAs (7+ characters) and bare URLs. Backticks render as inline code with no
> link.

### Step 6 — Resolve each addressed thread

Only after the reply has posted successfully, resolve the thread using the
GraphQL `resolveReviewThread` mutation.

If resolution fails, surface it. The reply already exists so the trail is
intact, but the user may need to resolve manually.

### Step 7 — Sync the PR description

After replies and resolutions land, check whether the PR description or any
AI-authored in-repo docs are stale:

- New behavior from fixed commits should be reflected.
- Dismissed items should be briefly noted so the description and diff don't
  disagree.
- Claims that are no longer true should be corrected.

Apply the authorship rule to decide how to update. If nothing is stale, this
step is a no-op — say so briefly and move on.

### Step 8 — Re-query and loop

Go back to step 1. Don't assume the unresolved set has only shrunk by what you
addressed — reviewers may have added new threads while you were working. The
fresh query is authoritative.

## Termination

Stop when:

- The step 1 query returns empty (after excluding deferred threads), or
- The user asks to stop, or
- Interactive review becomes unavailable (rule 2).

If tackled commits were pushed this session, suggest a re-review before the
final summary (see below). Skip for dismissals-only sessions or autonomous-mode
termination.

Summarize the session: commits pushed (with SHAs), threads replied to, threads
resolved, threads deferred, PR description or doc edits made (noting which were
user-approved vs. agent-owned), re-review requests sent, and any failures. Link
the PR.

## Suggesting a re-review

When the loop terminates with at least one tackled commit pushed, suggest
requesting a re-review.

### Default reviewer set

Build it as the union of:

- Authors of the originating comment on each tackled thread, excluding the PR
  author and any identity you've been posting under.
- Currently-requested reviewers who haven't yet submitted a review.

Present the default set and let the user confirm, edit, or skip.

Once confirmed, use `gh pr edit --add-reviewer` to send the requests.

### GitHub-specific notes

- `gh pr view --json reviewRequests` silently drops Bot-typed reviewers. To
  include bots and teams, query `reviewRequests` via GraphQL with explicit
  `... on User`, `... on Bot`, and `... on Team` type fragments.
- The REST `/requested_reviewers` endpoint rejects bot logins with 422. If
  `gh pr edit` fails for a bot, retry with just that bot's login via
  `gh pr edit`; don't fall back to REST for bots.
- `gh pr edit --add-reviewer` works for both first-time requests and re-requests
  from reviewers who already submitted.
