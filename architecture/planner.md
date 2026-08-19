# Planner

The planner turns obligations into time. It lives in [scalar-app/ai](https://github.com/scalar-app/ai) (`src/planner`) and is a pure function:

```ts
plan(request: PlanningRequest): PlanningResult
```

No database, no network, no model, no clock of its own: `now` is an input. That is deliberate. Scheduling is the part of Scalar most likely to be wrong in a way that costs someone a deadline, so it is the part that has to be reproducible and testable without standing anything up.

## AI interprets, the planner decides

```text
"find me two hours tomorrow afternoon"
        ↓  AI interpretation
{ intent: schedule_task, minutes: 120, preferredWindow: 12:00-18:00 }
        ↓  planner
proposed blocks, unscheduled items, conflicts, warnings
        ↓  person approves
persisted
```

A model may read a sentence and say what someone meant. It never works out whether an hour is free. See [ai-safety.md](ai-safety.md).

## Rules that do not bend

- Fixed blocks are never moved or removed. They are the shape of the day, not a suggestion.
- Nothing is proposed before `now`.
- Work never ends after its deadline. A task that cannot fit before its deadline comes back as unscheduled rather than as a block that quietly misses it.
- Everything that could not be placed comes back with a reason. Silence would be the worst outcome.
- The same request produces the same plan. Ties break on id, because "apply" has to mean the plan that was previewed.

## How it places work

1. Report overlaps among fixed blocks, then plan around them anyway. Someone double booked at ten is a fact about their day; refusing to plan the rest of it would not help.
2. Order tasks: due within a day, then within a week, then later, then undated. Priority within that, then the shorter job, then id.
3. Reorder so dependencies come before what needs them. A cycle is refused rather than guessed at.
4. Compute available windows: working hours in the person's zone, on their working days, minus everything already spoken for, with the buffer applied around each busy interval.
5. For each task in order, walk the days and place it in the first window it fits. Inside a day, the preferred part of the day wins; between days, sooner wins. A preference should not turn into a delay.
6. Add each proposal to the busy set, so later tasks plan around earlier ones.

Step 5 is the whole heuristic. It is not optimal and is not trying to be. It is explainable, which matters more, because a person has to be able to disagree with a specific decision.

## Explanations

Every proposed block carries machine readable reasons, which the UI phrases:

`due_within_24_hours`, `due_soon`, `high_priority`, `earliest_available`, `fits_available_window`, `preferred_focus_period`, `before_deadline`, `after_dependency`

> Scheduled Wednesday morning because it is due Wednesday afternoon and fits your preferred focus hours.

## Conflicts

Returned rather than resolved, because each one has a different answer:

| Kind | What it means |
| --- | --- |
| `overlapping_fixed_blocks` | Two things that cannot move are booked at once |
| `insufficient_time_before_deadline` | There is not enough free working time before it is due |
| `outside_working_hours` | There are no working hours in the range at all |
| `task_too_large_for_window` | The work is bigger than any free block |
| `dependency_incomplete` | Something it depends on could not be scheduled, or the dependencies form a cycle |

Warnings are softer: `placed_outside_preferred_window`, `no_estimate_used_default`, `deadline_in_the_past`.

## Time zones

All local reasoning goes through `ai/src/time.ts`, which is also what `findFreeSlots` uses. Working hours are wall clock times in the person's zone, so nine to five is eight hours on a normal day and still nine to five on the day the clocks change. Days are walked one at a time rather than by adding 24 hours, which is what keeps that true.

## The API around it

`POST /api/v1/planner/preview` returns a plan and writes nothing. `POST /api/v1/planner/apply` takes the approved blocks back and writes them in one transaction, refusing with `409 PLAN_STALE` if the day changed underneath the preview. See [api/v1/planner.md](../api/v1/planner.md).

Apply takes the blocks rather than re-planning, because what a person approved is a specific set of times, not whatever the planner would say a minute later.

## Not here yet

Splitting a task across several blocks, so a six hour job can use three two hour gaps. Today it is placed whole or not at all, which is the largest gap. After that: energy or context aware placement, re-planning work that already has a time, and durations learned from focus sessions.
