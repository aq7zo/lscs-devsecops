---
name: review
description: Review teammates' pull requests in LSCS repositories with parallel read-only subagents - pick the open PRs to review, give each one a reviewer subagent, spot-check the findings, then show the user every draft review together. Nothing is posted by a subagent, and Claude never approves. Use when the user says "review", "review the team's PRs", names PRs to review, or asks to review several PRs at once.
---

# Review: teammates' PRs in parallel

For PRs by others. The user's own PRs go through `pr`; nobody approves their own. Subagents can't ask the user anything, so they only do Tier 1 work: reads, a temp clone, local checks, returning a draft. Every Tier 3 step stays with you, in this session. Approving is Tier 4: the user clicks Approve.

## 1. Pick the PRs
List open PRs in the repos from the project facts (`gh pr list -R <owner/repo> --state open`), and keep the ones that:
- aren't by the user, and aren't drafts (unless the user names them);
- ask the user for review, or the user names them, or the project facts say the user reviews that repo;
- have no draft in `Tickets/<repo>/reviews/` for the current head SHA. A draft for an older head: review only the commits since.

Stacked PRs (one carries another's commits): review the base PR in full, and the others only from the base's head on. Base first in the list.

Cap: 4 PRs, unless the user names a number. The user must understand every review posted under their name. Show the list, one line per PR (author, branch, size, CI state, stacked on), plus every skipped PR and why. Start on the user's OK.

## 2. Review: one subagent per PR, all launched together
Give each reviewer a self-contained prompt. It doesn't see this conversation:
- the repo, PR number, title, author, head → base, head SHA, the stacked-on PR and its head SHA (or none), the linked issue, and the project facts line for the repo;
- **Do:** load `lscs-devsecops:conventions`. Read the PR, its diff, reviews and review comments with `gh`. Clone the head into `<temp>/<repo>-pr<N>` and run every check this machine can (install, build, lint, tests, dependency audit). Check:
  - correctness: bugs, error handling, edge cases;
  - security (CQA): secrets, input validation, authn/authz, injection, unsafe defaults, dependency advisories;
  - fit with the deploy setup: the Dockerfile, `.dockerignore`, runtime image, env vars documented in `.env.example`;
  - workflows and images, when the diff has them: action pinning, permissions, triggers, base images;
  - conventions: branch, Conventional Commits, PR body sections, assignee, scope against the linked issue;
  - process: CI, dismissed or stale approvals, open threads.
  Dependabot PRs: the changelog between versions, breaking changes, CI. Nothing more.
- **Don't:** post, comment, approve, request changes, push or edit anything on GitHub. Write no files outside the temp clone. Never print or handle a secret value: a secret found goes at the top of the return as name, file and line only. On a permission denial, stop and report it.
- **Return:** a suggested verdict (approve, comment, request changes) with one line why; findings labelled BLOCKING, QUESTION, SUGGESTION or NIT (`conventions`), each with `file:line` at the head SHA, the problem, a fix, and the rule; the checks run with pass or fail; what it couldn't verify.

## 3. Spot-check
Before anything reaches the user, read each BLOCKING finding against the diff yourself, and a sample of the rest. Drop findings that don't hold, and say how many you dropped. A suggested approve with a failing check, or a secret found, is reported to the user first.

## 4. Record and propose
Only this session writes notes:
- `Tickets/<repo>/reviews/pr-<N>-draft.md`: PR, head SHA, date, suggested verdict, findings, checks, open questions.
- HANDOFF: a `## Reviews` table (PR, author, head SHA, verdict, status) under the queue; add it if missing.

Then show every review together, each with: PR, head SHA, verdict, the exact review body (2–3 sentence summary first, findings after), and each inline comment with its `file:line`. The user approves, changes or holds each one separately.
- **Comment or request changes:** on the OK, post exactly as shown as one review pinned to that head SHA (`gh api repos/<owner>/<repo>/pulls/<N>/reviews` with `commit_id`, `event`, `body`, `comments`).
- **Approve:** Tier 4. Give the user the PR link and the exact approval text to paste; they submit it. Record it once it shows on GitHub.

## 5. Follow-up
At each `start`, re-read the PRs in the Reviews table: new commits since the reviewed SHA mean a delta review of those commits; replies to the user's comments get a drafted answer (Tier 3). Merged or closed: record who and when, and drop the row at the next `end`.
