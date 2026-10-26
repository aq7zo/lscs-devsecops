---
name: batch
description: Work several LSCS tickets at once with subagents - pick the ready tickets that don't depend on each other, give each one an implementer subagent, review each result with parallel read-only reviewers, then show the user every PR proposal together. Nothing is pushed by a subagent. Use when the user says "batch", "work these tickets in parallel", or asks to take on several tickets at once.
---

# Batch: several tickets in parallel

Run after `start` (this session or a recent one): it needs a fresh queue and fresh *Needs before start*. Subagents can't ask the user anything, so they only do Tier 1 work: reads, local files, local checks, commits in temp clones. Every Tier 3 step stays with you, in this session, through `pr`.

## 1. Pick the batch
From the HANDOFF queue, in rank order, take tickets that:
- have every need met (checked today, or re-check it live now);
- have a plan, or are small enough to plan from the ticket body without a decision only a person can answer;
- don't depend on each other, and don't touch the same file in the same repo (two tickets on one workflow file go one after the other);
- aren't waiting on review, and aren't ready for `close` (it stays in this session, one ticket at a time).

Cap: 3 tickets, unless the user names a number. The user must understand every change (CCG §16), so more tickets means more to review. Show the user the batch, one line per ticket, plus every skipped ticket and why. Start on the user's OK.

## 2. Implement: one subagent per ticket, all launched together
Give each implementer a self-contained prompt. It doesn't see this conversation:
- the ticket MD's path, the project facts line for its repo, and the branch name and base (`conventions`, `pr`);
- **Do:** load `lscs-devsecops:conventions`. Draft files in `deliverables/<repo>/` (only this ticket's files). Clone the base branch into `<temp>/<repo>-<N>`, copy the files in, create the branch, run every check this machine can (build, lint, tests, YAML parse, secret scan) and commit there with a Conventional Commits title.
- **Don't:** push, open or edit a PR, comment on GitHub, or edit `HANDOFF.md`, ticket MDs or `CLAUDE.md`. Don't commit in the notes repo. Never handle a secret value. On a permission denial or blocked step, stop and report it. Don't work around it.
- **Return:** clone path and branch; each file with its +/− lines; the checks run and their output; checks this machine couldn't run; each acceptance criterion and how the change meets it; open questions; a draft PR title and body from the repo's template.

## 3. Review: parallel read-only reviewers per finished ticket
When an implementer returns, launch these at once on its clone and diff. They read only and return findings (file, line, issue, which rule):
- **Conventions:** load `lscs-devsecops:conventions`. Check the branch, commit, PR body, scope (every line traces to the ticket) and CCG §9 author checks.
- **CI and container security:** action pinning, workflow permissions, triggers, secrets handling, images (CQA). Skip it when the diff has no workflow, Dockerfile or Compose file.
- **Acceptance:** each criterion, verbatim from the ticket, against the diff: met, not met, or can only be proven live after the merge.

Send real findings back to the same implementer once (continue it, don't start a new one). Findings it still can't fix go to the user as open questions.

## 4. Record and propose
Only this session writes notes, one ticket at a time:
- each ticket MD: status IN PROGRESS, delivery line (branch, clone), plan, checks and their output, review findings, open questions, a log line;
- HANDOFF: queue rows, and a to-do for each check this machine couldn't run.

Then show every PR proposal together, each in the full `pr` format (repo, base ← branch, title, exact body, reviewers, files with +/− lines), plus its review findings and open questions. The user approves, changes or holds each one separately. Push and open only the approved ones, exactly as shown, through `pr`. Held tickets keep their clone and branch, recorded in the MD.

Finish with the `end` PR brief for every proposal shown: 3 lines each (what it does, why, what could break).
