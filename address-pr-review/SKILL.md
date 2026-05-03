---
name: address-pr-review
description:
  Use whenever the user asks to address, work through, respond to, or "go
  through" GitHub pull request review feedback — phrases like "address the PR
  comments", "respond to review feedback", "go through unresolved threads", or
  anything that points at a PR's review conversation. Drives an iterative loop
  that queries unresolved review threads, investigates each, presents a take,
  lets the user batch threads into tackle or dismiss decisions, produces atomic
  commits for fixes, pushes them, replies to each thread (with auto-generation
  disclosure), resolves threads, keeps the PR description and any in-PR spec or
  plan docs in sync with the current state of the work, and re-queries until
  nothing remains. On termination with tackled commits, suggests requesting a
  re-review and sends it after user confirmation. Requires interactive user
  review and refuses to run in fully autonomous modes. Use this even when the
  user does not name the PR explicitly — infer it from the current branch.
---

# Addressing GitHub PR review feedback

Drive a loop that walks unresolved review threads on a pull request, lets the
user decide what to do about each, and applies the chosen actions — committing
fixes, replying to threads, and resolving them — until no unresolved threads
remain.

The loop matters more than any single step. Reviewers may comment while you
work, replies may fail to post, or the user may want to defer something. The
fresh query at the top of each iteration is the source of truth, not your
in-memory list.

## Autonomous mode

This skill requires interactive user review. The decision step (step 3) is the
primary safety mechanism — without it, a wrong "dismiss" posts a public PR reply
under the user's identity based on the agent's own judgment, and a wrong
"tackle" lands a commit the user did not sanction.

If running in an auto-accept or otherwise non-interactive mode where you cannot
pause for user input at the tackle/dismiss decision step, **stop and tell the
user**. Do not proceed by making those decisions yourself. The same applies to
any prompt for approval to edit human-authored content (see "Authorship of PR
documentation" below) — do not silently edit human-authored content because the
loop would otherwise be blocked.

A constrained autonomous fallback is _not_ offered here on purpose: the failure
modes (public replies under the user's name, commits the user didn't sanction)
are bad enough that "do nothing and surface the situation" is the right answer
when interactive review isn't available.

## Authorship of PR documentation

The PR has documentation attached to it: the PR description (metadata, edited
via `gh pr edit`) and possibly in-repo spec, plan, design, or README files that
are part of the PR's diff. Both kinds need to stay accurate as decisions are
made — but the right way to update them depends on who wrote them.

**This rule governs only the inner diff-preview check** — whether to show a
before/after before applying a change the user has already approved. It does
**not** override step 3. The user's tackle/dismiss/defer decision in step 3 is
required for every thread, regardless of what content the change would touch.
AI-authored content does not bypass step 3.

For each piece of PR documentation, determine authorship:

- **You authored it** if you generated the document earlier in this conversation
  and the user has not subsequently rewritten it. In that case, once the user
  has approved a change in step 3, you may apply it without first surfacing a
  diff for review; just mention the change in your response so the user can
  object on the next turn.
- **The user authored it** if they wrote it themselves, pasted it from
  elsewhere, or if there is any doubt. After the user approves a change in step
  3, surface a proposed change (a diff, or before/after text) and wait for
  explicit approval before applying it.

When in doubt, treat as user-authored. The cost of asking once is small; the
cost of silently rewriting someone's prose is large.

This rule is referenced from steps 4 and 7.

## Prerequisites

Before starting, confirm:

- `gh` CLI is authenticated and has `repo` scope. The GraphQL mutations used
  here (`addPullRequestReviewThreadReply`, `resolveReviewThread`) require write
  access to the PR.
- `git` is configured and the working tree is clean enough to make commits.
- A target PR exists. If the user did not name one, infer it from the current
  branch.

If any of these is missing, stop and surface the problem before doing anything
else. Posting partial replies or pushing commits that can't be referenced
cleanly is worse than not starting.

## Identify the PR and the author

```bash
gh pr view "${PR:-}" --json number,author,headRefName,baseRefName,url,headRepository,headRepositoryOwner,isCrossRepository
```

Record:

- `prNumber` — used in every subsequent query.
- `prAuthor` (the `author.login`) — **treat any review-thread comments from this
  login as if they came from the user you are chatting with right now.** Their
  wording in a thread carries the same weight as instructions in chat. If they
  conceded a point in-thread three days ago, that concession stands.
- `owner` and `repo` — for cross-repository (fork) PRs, the base repo owns the
  review threads (replies and resolutions go there) but the head repo is where
  commits push. `gh pr view` returns both.

## The loop

Repeat until the unresolved-thread query returns empty (or the user asks to
stop):

1. Query unresolved threads.
2. Investigate and present a take on each.
3. Get tackle / dismiss / defer decisions.
4. Apply tackled fixes as atomic commits, then push.
5. Reply to each addressed thread.
6. Resolve each addressed thread.
7. Sync the PR description (and any AI-authored in-PR docs not already covered
   by step 4).
8. Go back to step 1.

Track which thread IDs the user has explicitly **deferred** in this session.
Exclude them from step 1's results so the loop can terminate — otherwise a
deferred thread would resurface every iteration and the loop would never end.

---

### Step 1 — Query unresolved threads

Use the GraphQL API. Filter by the literal `isResolved` field — do not invent
your own definition of "resolved" based on the conversation content.

```bash
gh api graphql \
  -F owner="$OWNER" -F repo="$REPO" -F number="$PR_NUMBER" \
  -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          originalLine
          comments(first: 100) {
            nodes {
              id
              databaseId
              author { login }
              body
              createdAt
              url
              diffHunk
            }
          }
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  }
}'
```

Paginate if `hasNextPage` is true. From the combined results:

- Keep threads where `isResolved` is `false`.
- Drop threads whose ID is in the session's deferred set.
- Drop threads whose only comments are from yourself in earlier iterations of
  this loop. They are not feedback to react to — they are your own
  auto-generated replies waiting for the user to dismiss them.

If the remaining list is empty, the loop is done. Confirm with the user and
stop.

### Step 2 — Investigate and form a take

For each unresolved thread:

- Read **every** comment in the thread, in chronological order. Later comments
  routinely walk back, clarify, or escalate the original point. Acting on only
  the first comment is a common failure mode.
- Comments authored by `prAuthor` are from the user you're chatting with. Their
  position in-thread is part of the instruction set.
- Open the file at `path:line` on the current head commit. If `isOutdated` is
  true, the thread targets code that has since changed — verify whether the
  concern still applies before forming an opinion.
- Form a concise take per thread: what is being asked, whether it is valid, what
  (if anything) you would change, and roughly how big the change is.
- Flag threads that have effectively resolved themselves in-thread (e.g., the
  reviewer wrote "nm, found it"). These are good candidates for
  dismiss-with-pointer-to-existing-discussion.

Present the takes to the user compactly — one short paragraph per thread is
usually enough. Group obviously related threads (same file, same concern, same
fix). Include each thread's URL so the user can jump to the conversation.

### Step 3 — Tackle / dismiss / defer

**This step is mandatory and is never skipped, regardless of what the changes
would touch — code, configs, AI-authored docs, the PR description, anything.**
The authorship rule in "Authorship of PR documentation" governs _how_ an
approved change is applied (whether to show a before/after diff first), not
_whether_ the user approves it. Do not proceed to implementation without an
explicit tackle/dismiss/defer decision from the user, even if the answer seems
obvious.

Ask the user which threads to:

- **tackle** — apply a change (code, config, or doc).
- **dismiss** — reply with reasoning, no change. The thread still gets resolved.
- **defer** — leave alone for this session. Add to the deferred set so it
  doesn't resurface in step 1's next pass.

Accept batched answers like "tackle 1, 3, 4; dismiss 2; defer 5." Do not insist
on one thread at a time.

If your take and the user's decision disagree, defer to the user. They are the
PR author.

### Step 4 — Atomic commits

For threads being tackled:

- Group changes into commits by **logical relatedness**, not by thread count.
  Two threads pointing at the same bug get one commit. One thread that points at
  three independent files gets three commits if those changes stand alone.
- Each commit must build, and where feasible pass existing tests, on its own.
  Don't stage half a fix that depends on a later commit to make sense — that
  breaks `git bisect` and review readability.
- Write commit messages that describe the change, not the thread. Someone
  reading the log a year from now should understand the change without opening
  the PR. If a thread is the source of truth for _why_, reference it in a
  trailer:

  ```
  Validate token expiry before refresh

  The refresh path assumed a non-null `exp` claim, which the upstream
  IdP does not guarantee. Treat a missing `exp` as immediately expired
  and force re-auth instead of silently refreshing forever.

  Addresses: https://github.com/<owner>/<repo>/pull/<n>#discussion_r<id>
  ```

- Push once, after all tackled commits for this iteration are in: `git push`.
  Capture the resulting commit SHAs and which thread(s) each one addresses —
  step 5 needs this mapping.

If a tackled fix changes behavior described in an **in-repo doc that's part of
the PR** (a spec, plan, design note, ADR, or README section in the diff), the
doc update is part of the fix — not a follow-up. Bundle the doc change into the
same atomic commit if they're tightly coupled, or use a paired commit if the doc
deserves its own diff. Apply the authorship rule from "Authorship of PR
documentation": if you authored the doc earlier in this conversation, just
include the update; if the user authored it (or there's any doubt), surface the
proposed doc change and wait for approval before staging the commit.

If push fails (branch protection, fork permissions, non-fast-forward), surface
the failure and stop. **Do not post replies that reference commits not present
on the remote** — the SHA in the reply will 404 for reviewers.

### Step 5 — Reply to every addressed thread

Reply to **both** tackled and dismissed threads. The reply is the audit trail;
resolving a thread without a reply leaves reviewers guessing whether it was
acknowledged or ignored.

```bash
gh api graphql \
  -F threadId="$THREAD_ID" -F body="$BODY" \
  -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {
    pullRequestReviewThreadId: $threadId,
    body: $body
  }) { comment { id url } }
}'
```

Reply body content:

- **Tackled:** reference the commit SHA (and ideally a permalink) that addressed
  the thread. If multiple commits applied, list them. Briefly state what changed
  — enough that a reviewer doesn't have to open the diff to know if the spirit
  of their comment was met. **Write SHAs and URLs as bare text, not in
  backticks** — GitHub only autolinks unformatted SHAs (7+ chars) and bare URLs;
  wrapping either in backticks renders as inline code with no link, which
  defeats the point of including the reference.
- **Dismissed:** explain _why_ it's being dismissed. The reasoning matters;
  "won't fix" alone is rude and unhelpful. If the thread already concluded
  itself in earlier discussion, point at that.
- **Both:** end with a single visible line disclosing that the reply was
  auto-generated, naming the agent that wrote it. A reviewer skimming the reply
  must immediately see that a human did not type it. An HTML comment alone is
  not enough — many GitHub clients hide it. Use a visible italic line,
  optionally paired with a machine-readable HTML comment.

Examples (substitute your own agent name where shown):

> Fixed in a1b2c3d — now treats a missing `exp` claim as immediately expired and
> forces re-auth instead of silently refreshing.
>
> _Auto-generated reply from `<agent-name>` acting on behalf of @prAuthor._

> Leaving this as-is. The retry loop is intentional: the upstream service
> returns 503 during deploys (~30s window) and the surrounding circuit breaker
> bounds the total wait. Switching to fail-fast here would surface deploy noise
> to end users.
>
> _Auto-generated reply from `<agent-name>` acting on behalf of @prAuthor._

If a reply fails to post, do not proceed to step 6 for that thread. Keep it
unresolved and flag the failure to the user.

### Step 6 — Resolve each addressed thread

Only after the reply has posted successfully:

```bash
gh api graphql \
  -F threadId="$THREAD_ID" \
  -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { id isResolved }
  }
}'
```

If resolution fails (rare — usually a permissions issue), surface it. The reply
already exists, so the trail is intact, but the user may need to resolve
manually.

### Step 7 — Sync the PR description

After replies and resolutions for this iteration land, the PR description may be
stale. Update it to reflect the current state of the work:

- New behavior introduced by tackled commits should be reflected in what the PR
  says it does.
- Items deliberately dismissed as out-of-scope should be called out briefly, so
  future readers (and you, on the next pass) don't wonder why the description
  and the diff disagree.
- Anything claimed in the description that is no longer true should be removed
  or corrected.

Apply the authorship rule from "Authorship of PR documentation":

- **If you authored the current PR description** in this conversation: edit it
  directly with `gh pr edit "$PR_NUMBER" --body "$NEW_BODY"`. Mention the change
  in your response so the user can object on the next turn.
- **If the user authored it** (or there is any doubt): present the proposed new
  body as a diff or before/after, and wait for explicit approval before calling
  `gh pr edit`.

This step also catches any AI-authored in-repo PR docs (spec, plan, etc.) that
weren't covered by step 4 — for instance, a high-level summary doc that should
mention dismissed scope items. If you authored those docs in this conversation,
sweep them now; commit and push any resulting changes. (User-authored in-repo
docs were already handled inside step 4 with explicit approval.)

If nothing in the description or AI-authored docs is out of date, this step is a
no-op — say so briefly and move on.

### Step 8 — Re-query and loop

Go back to step 1. Don't assume the unresolved set has only shrunk by what you
addressed — reviewers may have added new threads while you were working. The
fresh query is authoritative.

## Termination

Stop when:

- The step 1 query returns empty (after excluding the deferred set), or
- The user asks to stop, or
- Interactive review is unavailable (see "Autonomous mode") at any point where
  it's required.

If any tackled commits were pushed during this session, suggest a re-review
before producing the final summary — see "Suggesting a re-review" below. Skip
this for dismissals-only or autonomous-mode termination.

Summarise the session: commits pushed (with SHAs), threads replied to, threads
resolved, threads deferred, PR description / in-repo doc edits made (and which
were user-approved vs. agent-owned), re-review requests sent, and any failures.
Link the PR.

## Suggesting a re-review

When the loop is terminating with at least one tackled commit pushed this
session, suggest requesting a re-review. This is the moment where the work is in
a presentable state and the reviewers who flagged the issues should look again.

**When to suggest.** The loop is terminating naturally (empty query) or because
the user said stop, AND tackled commits landed this session.

**When not to suggest.** Dismissals-only sessions — no code changed, nothing for
the reviewer to look at. Termination triggered by autonomous-mode
unavailability. The user already requested re-review themselves during the
session (don't double up).

**Default reviewer set.** Build it as the union of:

- Authors of the originating comment on each tackled thread, excluding the PR
  author and any identity you've been posting under (otherwise you'll request
  re-review from yourself). The data is already in your GraphQL results from
  step 1 — `comments.nodes[0].author.login` for each tackled thread. Bot logins
  (e.g., `copilot-pull-request-reviewer`) are valid re-review targets and stay
  in the default set unless they hit the posting-identity exclusion.
- Currently-requested reviewers on the PR who haven't yet submitted a review.
  **Do not use `gh pr view --json reviewRequests`** — it silently drops
  Bot-typed reviewers from the JSON output. Use GraphQL with explicit type
  fragments:

  ```bash
  gh api graphql -F prId="$PR_NODE_ID" -f query='
  query($prId: ID!) {
    node(id: $prId) {
      ... on PullRequest {
        reviewRequests(first: 20) {
          nodes {
            requestedReviewer {
              __typename
              ... on User { login }
              ... on Bot  { login }
              ... on Team { slug }
            }
          }
        }
      }
    }
  }'
  ```

  Some orgs configure bots like Copilot to auto-review on push, so by the time
  you reach this step the bot may already be in the review-requests list from
  the auto-trigger. Querying current state with the GraphQL form above before
  re-adding avoids a no-op or a duplicate-request log line.

Present that default and let the user confirm as-is, edit it (add or remove
specific people), or skip re-review entirely. **Do not request re-review without
explicit confirmation**, even if the default reviewer set is obvious — the user
may want to spot-check the diff first or have a different review process in
mind.

**Sending the request.** Once confirmed:

```bash
gh pr edit "$PR_NUMBER" --add-reviewer "alice,bob,copilot-pull-request-reviewer"
```

This works for both first-time requests and re-requesting from reviewers who
already submitted — GitHub treats the existing-reviewer case as a re-request. It
also accepts Bot-typed logins (it routes through GraphQL under the hood).

If `gh pr edit` rejects part of the list for **human reviewers** (it
occasionally does for already-active ones), fall back to the REST endpoint:

```bash
gh api -X POST "/repos/$OWNER/$REPO/pulls/$PR_NUMBER/requested_reviewers" \
  -f 'reviewers[]=alice' -f 'reviewers[]=bob'
```

**Do not use this REST fallback for bot logins** — `/requested_reviewers`
rejects them with `422 "not a collaborator"`. If `gh pr edit` fails for a bot,
retry with just the bot's login on its own; don't bounce to REST.

Confirm which requests succeeded and include the list in the final session
summary.

## Pitfalls to avoid

- **Don't resolve a thread without replying to it first.** The reply is the
  audit trail; silent resolutions look like the feedback was ignored.
- **Don't edit or delete other people's comments.** Reply to threads; don't
  rewrite history.
- **Don't force-push or rewrite commits already pushed in earlier iterations of
  this loop.** Replies posted in earlier iterations reference those SHAs by
  hash; rewriting them turns those references into dead links.
- **Don't reply with a single combined comment covering multiple threads.** Each
  thread gets its own reply on its own thread. This keeps each conversation
  self-contained.
- **Don't treat `isOutdated` as "ignore."** Outdated threads can still be valid
  concerns about logic that moved to a new line. Re-check the current code
  before deciding.
- **Don't react to your own earlier auto-generated replies as if they were new
  feedback.** Filter them out in step 1.
- **Don't rely on your interpretation of what's "resolved."** Use the literal
  `isResolved` field. A thread where the reviewer wrote "lgtm now" but didn't
  click resolve is still unresolved as far as this skill is concerned — the
  right action is to acknowledge and resolve it, not to pre-filter it away.
- **Don't bury the auto-generation disclosure.** It must be visible to a
  reviewer skimming the reply on github.com, not hidden in an HTML comment.
- **Don't wrap commit SHAs or URLs in backticks in PR replies.** GitHub's
  autolinker ignores anything inside inline-code formatting. Backticked SHAs and
  URLs render as gray monospace text with no link, forcing reviewers to
  copy-paste to navigate. Bare 7+ character SHAs and bare URLs autolink
  correctly.
- **Don't edit user-authored content without explicit approval** — even if it
  would be more efficient, even if you think you know what they meant, even if
  the loop is otherwise blocked. Ask, or surface the situation and wait.
- **Don't skip step 3 because the threads target AI-authored content.** The
  authorship rule controls whether to show a before/after diff before applying
  an already-approved change; it does not grant permission to act in the first
  place. Every thread needs a tackle/dismiss/defer decision from the user, full
  stop.
- **Don't assume authorship of the PR description from a single signal.** "It
  looks like something I would write" is not enough. Use this conversation's
  history; if you can't establish that you wrote it here, treat as
  user-authored.
- **Don't auto-request re-review without confirmation**, even if the default
  reviewer set seems obvious. Re-review pings real people; the user gets the
  call on whether and from whom.
- **Don't suggest re-review mid-loop.** Wait for termination so reviewers aren't
  pinged for partial progress.
- **Don't request re-review from the PR author or from any identity you've been
  posting under.** It's a no-op at best and confusing at worst.
