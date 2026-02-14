## on_pr_reviewed — Issue Doc

### Summary

Implement a GitHub Actions workflow `on_pr_reviewed` for `oppia/oppia` that enforces reviewer-related policies when pull request reviews are submitted.

This workflow should handle two main cases:

- Review is an approval:  
  a) If there are pending reviews, assign the reviewers who haven't yet reviewed the PR and ping them.  
  b) If all reviewers have approved, add the `PR: LGTM` label and assign the PR author or a project owner to merge the PR.
- Review requests changes: unassign the reviewer, assign the PR author, and remove the `PR: LGTM` label if present.

### Background / Motivation

This replaces Oppiabot functionality for handling review actions and reduces manual triage. It should integrate with existing labels and teams (e.g. `@oppia/project-owners`) and follow the behavior described in the project planning spreadsheet.

### Scope / Non-goals

- Scope: Implement `on_pr_reviewed` workflow in `oppia/oppia` as a GitHub Actions workflow (or small action + workflow) that triggers on `pull_request_review` events. Add tests for its logic. Provide a clear implementation doc and instructions for running locally / in CI.
- Non-goals: Implementing unrelated Oppiabot features (CLA checks, force-push handling, other label workflows) — those will be implemented separately.

### Acceptance Criteria

1. When a review with state `approved` is submitted:
   - If there remain pending reviewer approvals (i.e., some requested reviewers haven't yet approved), assign the remaining reviewers and post a polite ping comment for their review.
   - If all requested reviewers have approved, add the `PR: LGTM` label (or `LGTM` depending on existing naming) and assign the PR author or a project owner to merge the PR.
2. When a review with state `changes_requested` is submitted:
   - Unassign the reviewer who requested changes.
   - Assign the PR author.
   - Remove the `PR: LGTM` label if present.
3. The workflow must not act on reviews from bots or specific excluded actors.
4. The workflow must log actions and create clear comments explaining its changes (assignments/label changes) so authors and reviewers understand the automation.
5. Tests cover the main code paths (approval with pending reviewers, approval with all reviewers approved, changes requested).

### Implementation Plan

1. Create a new workflow file `./github/workflows/on_pr_reviewed.yml` that triggers on `pull_request_review` (types: submitted).
2. Implement a small JS/TS action (or reuse a shared action if available) to encapsulate logic so we can unit-test it:
   - `actions/on-pr-reviewed/src/index.js` (or TypeScript) — reads event payload, queries GitHub API for requested reviewers and current review states, performs the actions (assign, unassign, add/remove label, comment).
   - Use `GITHUB_TOKEN` with appropriate permissions; restrict destructive operations to CLA-signed contributors and org members as per repo policy.
3. Tests:
   - Unit tests for the action logic (simulate payloads and stub GitHub API calls).
   - Integration test (optional): a GitHub Actions workflow run on a test repository or using `act` locally.
4. Add a small helper module `lib/checkPullRequestReview.js` (or reuse existing `lib/checkPullRequestReview.js` if present in Oppiabot code converted to the action) adapted for Actions environment.
5. Add workflow comment templates and ensure messages are friendly and actionable.
6. Add documentation: this file and an implementation README in `.github/actions/on-pr-reviewed/README.md`.

### Files to Create / Modify

- Create workflow: `.github/workflows/on_pr_reviewed.yml` (calls the action).  
- Add action source: `.github/actions/on-pr-reviewed/action.yml` and `.github/actions/on-pr-reviewed/src/*`  
- Add unit tests under `actions/on-pr-reviewed/__tests__` or repo `core/tests` depending on preferred test harness.  
- Add README: `.github/actions/on-pr-reviewed/README.md`.

### Key Design Details

- Determining pending reviewers:
  - Use the pull request `requested_reviewers` + `requested_teams` plus the review history to find who still hasn't submitted an approval.
  - Consider reviewers who explicitly requested changes as not approved.
- Assigning a reviewer who hasn't yet reviewed: call GitHub `issues.addAssignees` with the reviewer username(s).
- When all reviewers have approved: add label `PR: LGTM` (or `LGTM`) and assign the PR author or a member of `@oppia/project-owners` (choose PR author first, fallback to a random project owner).
- When a reviewer requests changes: unassign reviewer via `issues.removeAssignees` and assign PR author via `issues.addAssignees`.
- Prevent loops: include a marker in comments (e.g., hidden HTML comment) to avoid the workflow triggering itself repeatedly or re-processing the same review event.

### Comment / Message Text (examples)

- Approval with pending reviewers: "Thanks for approving! The following reviewers are still requested: @alice, @bob — assigning them for review."  
- All reviewers approved: "All requested reviewers have approved — adding `PR: LGTM` and assigning @author to merge.
If you prefer a project owner to merge, ping @oppia/project-owners."
- Changes requested: "@reviewer requested changes — unassigning them and assigning @author to address the requested changes. The `PR: LGTM` label has been removed."

### Tests

- Unit tests should mock GitHub API endpoints used: `GET /repos/:owner/:repo/pulls/:pull_number/reviews`, `GET /repos/:owner/:repo/pulls/:pull_number`, `GET /repos/:owner/:repo/issues/:issue_number/labels`, `POST /repos/:owner/:repo/issues/:issue_number/comments`, `POST /repos/:owner/:repo/issues/:issue_number/labels`, `POST /repos/:owner/:repo/issues/:issue_number/assignees`, `DELETE /repos/:owner/:repo/issues/:issue_number/assignees`.
- Test cases:
  1. Approval where one reviewer remains (ensure remaining reviewer assigned and pinged).
  2. Approval where all reviewers have approved (ensure `PR: LGTM` added and PR author assigned for merge).
  3. Review with `changes_requested` (ensure reviewer unassigned, PR author assigned, `PR: LGTM` removed).

### Permissions & Security

- The workflow will run using `GITHUB_TOKEN` and must be given permission to read and write PRs, issues, labels and collaborators. Add `permissions` block to workflow to scope tokens.
- Only act on human users (ignore bot accounts). Optionally enforce org membership for some actions.

### Rollout & Verification

1. Implement as a draft workflow and test on a fork/test repo.
2. Run unit tests in CI.
3. Enable in `oppia/oppia` on a limited basis (e.g., run on `pull_request_review` events from forks only) and monitor logs for a week.

### Implementation Notes / Risks

- Race conditions: multiple reviewers submitting simultaneously could result in duplicate assignments — ensure operations are idempotent.
- Label name consistency: confirm exact label name used in repo (e.g., `PR: LGTM` vs `LGTM`). Use the repo's existing label names.
- Avoid spamming with comments — aggregate messages where appropriate.

### Next Steps (immediate)

1. Implement the action skeleton under `.github/actions/on-pr-reviewed/` and add unit tests.  
2. Create `.github/workflows/on_pr_reviewed.yml` to run the action on `pull_request_review` events.  
3. Open a PR with the implementation and tests; request review from `@oppia/server-admins-team` and `@oppia/project-owners`.

---
If you'd like, I can now scaffold the action and the workflow files and add the unit tests in this repo. Tell me to proceed and I'll create the files and tests.
