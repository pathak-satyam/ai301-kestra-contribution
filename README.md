# AI301 Capstone — Kestra Contribution

- **Contributor:** Satyam Pathak ([@pathak-satyam](https://github.com/pathak-satyam))
- **Project:** [Kestra](https://github.com/kestra-io/kestra), an event-driven orchestration and scheduling platform (Java backend, Vue 3 frontend)
- **Issue:** [#8337 — Align the bulk and single execution actions for both Replay and Restart](https://github.com/kestra-io/kestra/issues/8337)
- **Pull request:** [#16900 — feat(executions): align bulk restart revision picker with bulk replay](https://github.com/kestra-io/kestra/pull/16900)
- **Status:** ✅ **Merged** into `develop` on July 29, 2026 — approved by the frontend and backend maintainers, closing issue #8337.

## Overview

Kestra lets you re-run failed executions two ways, Replay and Restart, and each can run on a single execution or in bulk across many. When you re-run, you can use the flow's original revision or its latest revision.

Before my change, three of those four paths already let you choose the latest revision. Bulk Restart was the exception: it always used the original revision, with no way to pick the latest. So the behavior looked like this:

| Action  | Single execution     | Bulk execution         |
|---------|----------------------|------------------------|
| Replay  | original and latest  | original and latest    |
| Restart | original and latest  | original only          |

My PR added the missing option to bulk Restart across the whole stack (REST API, store, UI, tests) so all four paths behave the same way.

## Phase I — Issue selection

I picked this issue because the maintainers had marked it as a good first issue and because it was well scoped. The behavior I needed to add already existed for bulk Replay in the same codebase, so I had a working reference to follow instead of designing something from scratch. That made it easy to say up front what "done" would look like:

1. The bulk Restart endpoints accept an optional flag to use the latest flow revision.
2. Clicking bulk Restart in the UI opens a confirmation dialog offering original vs. latest revision, matching the bulk Replay dialog.
3. There are tests for both the backend flag and the frontend store call.
4. The existing default behavior (original revision) still works.

I asked for the issue on June 17 and was assigned by the maintainer `@MilosPaunovic` on June 18, 2026.

## Phase II — Reproduce and plan

I set up the project locally with the backing services from `docker-compose-ci.yml` and ran the app at `localhost:8080`, with the frontend dev server from `ui/`.

To reproduce, I selected a few failed executions and clicked Restart. It ran immediately on the original revision with no dialog. Doing the same with Replay opened a dialog that let me choose the latest revision. That confirmed the reported inconsistency.

I traced why the two paths differed:

- In `ui/src/components/executions/Executions.vue`, the Replay button opened a dialog, but the Restart button called `restartExecutions()` directly through a plain confirm, with no revision choice.
- In `ui/src/stores/executions.ts`, `bulkRestartExecution` posted only the execution IDs and dropped any query params, so the UI couldn't send a `latestRevision` flag even if it wanted to.
- In `webserver/.../api/ExecutionController.java`, the bulk restart endpoints had no `latestRevision` parameter and always called `Restart.from(execution, null)`. The bulk replay endpoints already accepted the flag.

My plan was to add the capability from the bottom up so each layer could be tested on its own: backend flag first, then the store, then the dialog, with tests at the backend and frontend layers.

## Phase III — Build

I split the work into two commits for a cleaner history.

**Backend** (`ExecutionController.java`): I added an optional `latestRevision` query parameter (default `false`) to both `restartExecutionsByIds` (`POST /executions/restart/by-ids`) and `restartExecutionsByQuery` (`POST /executions/restart/by-query`), documented with an OpenAPI parameter. When the flag is true, I look up the flow and pass its current revision into the restart command:

```java
Integer revision = null;
if (Boolean.TRUE.equals(latestRevision)) {
    Flow flow = flowRepository
        .findById(execution.getTenantId(), execution.getNamespace(), execution.getFlowId(), Optional.empty())
        .orElseThrow();
    revision = flow.getRevision();
}
executionCommandQueue.emit(Restart.from(execution, revision).withOperationId(opId));
```

`restartExecutionsByQuery` just forwards the flag on to `restartExecutionsByIds`. The default of `false` keeps the old behavior, so this is not a breaking change. I added two `ExecutionControllerRunnerTest` cases covering bulk restart with the flag.

**Frontend** (`Executions.vue`, `executions.ts`): the bulk Restart button now opens a confirmation dialog instead of running immediately. The dialog offers Cancel, "restart latest revision" (`restartExecutions(true)`), and OK for the original revision (`restartExecutions(false)`), mirroring the bulk Replay dialog. I reworked `restartExecutions` to forward `{ latestRevision }`, and fixed the store's `bulkRestartExecution` so it actually sends the params to the API. I added three store unit tests in `ui/tests/unit/stores/executionsBulkRestart.spec.ts`.

I followed the project's conventions along the way: Conventional Commits with the `executions` scope, only `Ks*` design-system components in the UI (no raw Element Plus), i18n keys instead of hardcoded strings, and OpenAPI annotations on the endpoints.

## Phase IV — Submit and iterate

I opened PR #16900 against `develop` on June 18, 2026, referencing issue #8337. The change was reviewed in two parts: `@flcarre` approved the frontend on June 19 ("ok for front, thanks and nice work!"), and `@loicmathieu` approved the backend and merged it on July 29 ("LGTM, thanks a lot"), with `@MilosPaunovic` coordinating the review. Because the backend review took several weeks, I kept the branch current with `develop` throughout so it stayed mergeable. Merging the PR closed issue #8337. A full feedback log with dates and commit references is below.

After the change, all four re-run paths offer the same original-vs-latest revision choice:

| Action  | Single execution     | Bulk execution                 |
|---------|----------------------|--------------------------------|
| Replay  | original and latest  | original and latest            |
| Restart | original and latest  | original and latest (now fixed) |

## Maintainer Feedback Log

| Date | From | Feedback / event | My response |
|------|------|------------------|-------------|
| Jun 18, 2026 | — | Opened PR #16900 against `develop` with two commits: `4cdd300e` (backend `latestRevision` support) and `c1679dc7` (frontend revision-picker dialog). | Referenced `Closes #8337`; documented the change and test evidence. |
| Jun 19, 2026 | `@MilosPaunovic` | Triaged the PR, added labels `area/frontend` and `kind/external`, and requested a frontend review from `@flcarre`. | No change requested; waited for review. |
| Jun 19, 2026 | `@flcarre` | Approved the frontend: *"ok for front, thanks and nice work!"* | No changes needed; frontend accepted as-is. |
| Jul 2 & Jul 22, 2026 | `@MilosPaunovic` | Requested and then re-pinged `@loicmathieu` for the backend review. | Kept the branch current by merging `develop` (Jul 2 and Jul 27) so it stayed conflict-free and mergeable during the wait. |
| Jul 29, 2026 | `@loicmathieu` | Approved the backend: *"LGTM, thanks a lot"* and **merged** the PR into `develop`. | Merge closed issue #8337; verified the merged behavior against the acceptance criteria. |

No change requests were raised — both reviewers approved on the first pass — so the log reflects approvals plus the branch-maintenance work needed to keep a PR mergeable through a multi-week backend-review queue.

## Learnings & Reflections

**Technical gains.** The most useful thing I did was read the existing bulk Replay code before writing anything, so I could match it instead of guessing at a design. Tracing the bug through the Vue component, the Pinia store, and the Micronaut controller taught me how the frontend and backend hand off to each other — and the store quietly dropping the query params turned out to be the key detail that made the whole feature impossible from the UI. I also learned how a large project keeps API and UI in sync (OpenAPI annotations on the controller, generated client, i18n keys, and `Ks*` design-system components), and that matching those conventions is as much a part of a contribution as the code itself.

**What I'd do differently.** I'd surface the backend reviewer earlier. The frontend was approved in a day, but the backend sat for roughly six weeks waiting on a reviewer who had to be pinged twice. Next time I'd flag in the PR description exactly which files need backend eyes and proactively @mention a backend maintainer up front, rather than leaving reviewer routing entirely to the coordinator. I'd also add the before/after UI screenshot directly in the PR body instead of relying on the description of the dialog, to make the change reviewable at a glance.

**Teachable insight for future cohorts.** The hardest part of a "good first issue" wasn't the code — it was keeping a small PR alive across a long review queue. A change that's approved on the frontend can still stall for weeks on the backend, and a PR that isn't kept in sync with the fast-moving `develop` branch quietly rots into merge conflicts and gets deprioritized. The lesson: pick an issue that already has a working sibling pattern to copy, keep the diff small, and treat "keep the branch mergeable and the reviewers reminded" as part of the job — not something that happens automatically after you push.

## Links

- Issue: https://github.com/kestra-io/kestra/issues/8337
- Pull request (merged): https://github.com/kestra-io/kestra/pull/16900
- My fork: https://github.com/pathak-satyam/kestra
