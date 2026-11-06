# concurrency-probe

Does a job that is skipped by its own `if` still cancel an in-progress run
when `concurrency` has `cancel-in-progress: true`? Does it matter whether
`concurrency` is set on the **workflow** or on the **job**?

## Short answer

With a workflow that has a single job, which is skipped by `if` in the newer run:

- **Workflow-level** `concurrency`: the in-progress run ends as `cancelled`.
- **Job-level** `concurrency`: the in-progress run ends as `success`.

So even with only one job, the two placements are not equivalent as soon as
that job can be skipped.

## Setup

Two workflows, identical except for where `concurrency` sits
(see `.github/workflows/`). Each has one job that sleeps for 75 seconds and can be
skipped through the `skip` input (`if: ${{ !inputs.skip }}`).
The concurrency group includes a `trial` input, so many trials run in parallel
without interfering with each other.

Per trial:

1. Dispatch run **A** with `skip=false` (the job starts and sleeps).
2. Once every A job is in progress, dispatch run **B** on the same trial id:
   - `skip` trials: B has `skip=true`, so its only job is skipped.
   - `ctrl` trials (positive control): B has `skip=false`, so its job runs.

Three trials per cell.

## Results

| `concurrency` placement | B | trials | A conclusion | B conclusion |
|---|---|---|---|---|
| workflow | skipped by `if` | 3 | `cancelled` | `skipped` |
| workflow | runs (control) | 3 | `cancelled` | `success` |
| job | skipped by `if` | 3 | `success` | `skipped` |
| job | runs (control) | 3 | `cancelled` | `success` |

In the control trials, a B that actually runs cancels A under both placements,
so the cancellation mechanism itself behaves as expected.

### Runs

| placement | trial | A | B |
|---|---|---|---|
| wf-level | skip-1 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183077715) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183179252) |
| wf-level | skip-2 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183083635) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183184226) |
| wf-level | skip-3 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183091560) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183189751) |
| wf-level | ctrl-1 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183097472) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183194734) |
| wf-level | ctrl-2 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183103372) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183200851) |
| wf-level | ctrl-3 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183109775) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183206008) |
| job-level | skip-1 | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183114959) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183211873) |
| job-level | skip-2 | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183120236) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183217558) |
| job-level | skip-3 | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183125437) | [skipped](https://github.com/nemolize/concurrency-probe/actions/runs/42183222099) |
| job-level | ctrl-1 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183129564) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183227416) |
| job-level | ctrl-2 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183134372) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183231976) |
| job-level | ctrl-3 | [cancelled](https://github.com/nemolize/concurrency-probe/actions/runs/42183139878) | [success](https://github.com/nemolize/concurrency-probe/actions/runs/42183237329) |

## Observations and limits

- Observed on 2026-11-06 with GitHub-hosted `ubuntu-latest`; behaviour of GitHub Actions can change.
- The runs were started with `workflow_dispatch`, not `pull_request` or `push`.
- In every trial, A's job lasted about 77 seconds even when A ended as `cancelled`:
  the 75-second sleep was not cut short. The effect seen here is the recorded
  conclusion (`cancelled` vs `success`), not an early stop.
- Three trials per cell; every cell was unanimous.

## Why it matters

This experiment models, but does not reproduce, a common pattern: a workflow
that commits as a bot and guards re-entry with `if: github.actor != '<bot>'`
on its job. The bot's push starts a new run whose job is skipped. By the results
above, workflow-level `concurrency` would mark the run that did the push as
`cancelled`, while job-level `concurrency` would not. A real bot push was not tested.

## Reproduce

Dispatch one of the workflows twice with the same `trial`
(first with `skip=false`, then while it is running with `skip=true`),
and read the conclusion of the first run:

```sh
gh workflow run wf-level.yml -f trial=t1 -f skip=false
# wait until the job is in progress, then:
gh workflow run wf-level.yml -f trial=t1 -f skip=true
```
