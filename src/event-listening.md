---
outline: deep
title: Events & Observability
description: Subscribe to node-cron task lifecycle events (task started/stopped/destroyed and execution started/finished/failed/missed/overlap/maxReached), each carrying a TaskContext.
---

# Events & Observability

Once a task is running, you'll often want to *know* what it's doing: when it ran, whether it succeeded, how long it took, whether a run was missed. node-cron exposes this through lifecycle events.

Every `ScheduledTask`, whether inline or [background](/background-tasks), supports `.on()`, `.once()`, and `.off()`. Each listener receives a `TaskContext` with metadata about the task and the specific execution.

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', async () => {
  return doWork();
});

task.on('execution:finished', (ctx) => {
  console.log(`done in ${ctx.execution?.finishedAt - ctx.execution?.startedAt}ms`);
});

task.on('execution:failed', (ctx) => {
  console.error('failed:', ctx.execution?.error?.message);
});
```

> 💡 Attach listeners **before** the task starts to avoid missing early events. With `cron.schedule` the task starts immediately, so for guaranteed coverage create it stopped with [`cron.createTask`](/task-lifecycle#creating-a-stopped-task), attach listeners, then call `.start()`.

## Available events

| Event                  | Payload       | Emitted when…                                                    |
| ---------------------- | ------------- | ---------------------------------------------------------------- |
| `task:started`         | `TaskContext` | The task is started via `.start()`.                              |
| `task:stopped`         | `TaskContext` | The task is stopped via `.stop()`.                               |
| `task:destroyed`       | `TaskContext` | The task is destroyed via `.destroy()`.                          |
| `task:failed`          | `TaskContext` | A [background task](/background-tasks)'s daemon exited unexpectedly (crash, OOM-kill). `ctx.error` carries the reason. Inline tasks never emit this. |
| `execution:started`    | `TaskContext` | Right before the task function runs.                             |
| `execution:finished`   | `TaskContext` | The task function finishes successfully.                         |
| `execution:failed`     | `TaskContext` | The task function throws or rejects.                             |
| `execution:missed`     | `TaskContext` | A scheduled run was missed (blocking I/O or high CPU).           |
| `execution:overlap`    | `TaskContext` | A run was skipped because a previous one was still going (`noOverlap`). |
| `execution:maxReached` | `TaskContext` | `maxExecutions` was reached. The task is then destroyed.         |
| `execution:skipped`    | `TaskContext` | A [`distributed`](/distributed-coordination) run was skipped on this instance. `ctx.reason` is `'not-elected'` (another instance ran it) or `'coordinator-error'` (the coordinator failed; failed closed). |

## Subscribing

```js
import cron from 'node-cron';

const task = cron.createTask('* * * * *', async (ctx) => {
  console.log('running at', ctx.dateLocalIso);
  return 'done';
});

// React every time
task.on('execution:finished', (ctx) => {
  console.log('result:', ctx.execution?.result);
});

// React just once, then auto-remove
task.once('task:started', () => console.log('scheduler is up'));

// Stop listening
const onFail = (ctx) => console.error(ctx.execution?.error);
task.on('execution:failed', onFail);
task.off('execution:failed', onFail);

task.start();
```

## TaskContext

Every event delivers a `TaskContext`, giving consistent access to timing and execution metadata.

```ts
export type TaskContext = {
  date: Date;
  dateLocalIso: string;
  triggeredAt: Date;
  task?: ScheduledTask;
  execution?: Execution;
  reason?: 'not-elected' | 'coordinator-error';
  error?: Error;
};
```

| Field          | Type             | Description                                                           |
| -------------- | ---------------- | --------------------------------------------------------------------- |
| `date`         | `Date`           | The time the run was scheduled for.                                   |
| `dateLocalIso` | `string`         | Human-readable local timestamp, using the task's timezone.            |
| `triggeredAt`  | `Date`           | When the event was actually emitted. Useful for spotting drift.       |
| `task`         | `ScheduledTask?` | The task instance.                                                    |
| `execution`    | `Execution?`     | Details of the run (present for `execution:*` events).                |
| `reason`       | `string?`        | Why a run was skipped. Present only on [`execution:skipped`](/distributed-coordination#knowing-when-an-instance-skips): `'not-elected'` or `'coordinator-error'`. |
| `error`        | `Error?`         | The daemon's exit error. Present only on `task:failed`.               |

### Execution

`TaskContext.execution` describes a single run of the task:

```ts
export type Execution = {
  id: string;
  reason: 'invoked' | 'scheduled';
  startedAt?: Date;
  finishedAt?: Date;
  error?: Error;
  result?: any;
};
```

| Field        | Type      | Description                                                        |
| ------------ | --------- | ----------------------------------------------------------------- |
| `id`         | `string`  | Unique id for this execution.                                     |
| `reason`     | `string`  | `'scheduled'` (fired by the schedule) or `'invoked'` (via `execute()`). |
| `startedAt`  | `Date?`   | When the run started.                                             |
| `finishedAt` | `Date?`   | When the run finished.                                            |
| `error`      | `Error?`  | The error, if the run failed.                                     |
| `result`     | `any?`    | The return value, if the run succeeded.                           |

## Notes

- All listeners receive a `TaskContext`, even for non-execution events like `task:stopped` (where `execution` is absent).
- **Background tasks** emit the same events with the same context, relayed from the worker process. They also emit `task:failed` if their daemon dies unexpectedly; node-cron does not auto-restart, so react to it if you want to recover:

  ```js
  task.on('task:failed', (ctx) => {
    console.error('daemon died:', ctx.error?.message);
    task.start(); // restart manually
  });
  ```
- Listening to `execution:missed` also **suppresses** the default missed-execution warning, since node-cron assumes you're handling it. See [Logging](/logging#suppressing-the-missed-execution-warning).

## Next steps

- **[Background Tasks](/background-tasks)**: run jobs in isolated processes (same events apply).
- [Logging](/logging): route the warnings and errors behind these events through your own logger.
