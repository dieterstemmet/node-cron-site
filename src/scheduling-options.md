---
outline: deep
title: Scheduling Options
description: Fine-tune node-cron tasks with timezone, noOverlap, maxExecutions, maxRandomDelay, name, logger, and suppressMissedWarning, with examples for each.
---

# Scheduling Options

A bare `cron.schedule(expression, task)` already covers the common case. As your jobs grow more demanding, the optional third argument lets you tune *how* they run, without changing the rest of your code.

```js
cron.schedule(expression, task, options);
```

## All options

```ts
export type Options = {
  name?: string;
  timezone?: string;
  noOverlap?: boolean;
  maxExecutions?: number;
  maxRandomDelay?: number;
  logger?: Logger;
  suppressMissedWarning?: boolean;
  missedExecutionTolerance?: number;
  distributed?: boolean;             // run on one instance per fire across a fleet
  runCoordinator?: RunCoordinator;   // per-task coordinator (overrides the global one)
  distributedLease?: number;           // lease ms for lease-based coordinators (default 30000)
  unref?: boolean;                     // don't keep the process alive for this task's timer
  executeTimeout?: number; // background tasks only
  startTimeout?: number;   // background tasks only
};
```

| Option                  | Type      | Default | Description                                                                 |
| ----------------------- | --------- | ------- | --------------------------------------------------------------------------- |
| `name`                  | `string`  | task id | A human-readable identifier for the task. Useful for logging, debugging, and dashboards. |
| `timezone`              | `string`  | system  | The timezone the cron expression is evaluated in. Any IANA name recognized by `Intl.DateTimeFormat` (e.g. `"America/Sao_Paulo"`, `"UTC"`, `"Europe/London"`). See [Timezones & DST](/timezones-and-dst) for behavior across daylight-saving transitions. |
| `noOverlap`             | `boolean` | `false` | If `true`, a scheduled run is **skipped** when the previous run is still executing, preventing overlapping executions. |
| `maxExecutions`         | `number`  | none    | Maximum number of times the task may run. After the limit, the task is automatically destroyed. |
| `maxRandomDelay`        | `number`  | `0`     | Adds up to this many milliseconds of random delay (jitter) before each run. Spreads out tasks that would otherwise fire simultaneously. |
| `logger`                | `Logger`  | global  | A custom [logger](/logging) for this task, overriding the global one. **Not supported for [background tasks](/background-tasks).** |
| `suppressMissedWarning` | `boolean` | `false` | Silences the "missed execution" warning for this task. See [Logging](/logging#suppressing-the-missed-execution-warning). |
| `missedExecutionTolerance` | `number` | `1000` | How late (in ms) a scheduled run may wake and still execute instead of being reported as missed. Long timers drift (OS sleep, GC, throttling, clock skew), which can otherwise skip daily/weekly runs. Always capped to the gap to the next run, so it can never run a slot twice. |
| `distributed`           | `boolean` | `false` | Run this task on a **single instance per fire** across a fleet. Requires a `name`. Uses the `NODE_CRON_RUN` env-var default (one designated runner) unless a coordinator is set. See [Distributed Coordination](/distributed-coordination). |
| `runCoordinator`        | `RunCoordinator` | global | A per-task [run coordinator](/distributed-coordination#high-availability-a-custom-run-coordinator), overriding the one set with `setRunCoordinator`. Only used when `distributed` is `true`. |
| `distributedLease`        | `number`  | `30000` | Lease expiry (ms) passed to lease-based coordinators (e.g. a Redis lock) in case the holder crashes mid-run. Must exceed the run time (and, when combined with `maxRandomDelay`, the jitter too, the lease is taken before the jitter). See [Distributed Coordination](/distributed-coordination#distributedlease). Ignored by the env-var default. |
| `unref`                 | `boolean` | `false` | If `true`, the task's internal timer is [`unref`](https://nodejs.org/api/timers.html#timeoutunref)'d so it won't keep the Node.js process alive on its own. Handy for CLI tools and scripts that should exit once their real work is done. Toggle at runtime with [`task.unref()` / `task.ref()`](/task-lifecycle#unref-ref). |

> 🛈 There is no `scheduled` or `runOnInit` option anymore. Tasks created with `cron.schedule` start immediately; for a task that starts stopped, use [`cron.createTask`](/task-lifecycle#creating-a-stopped-task). To run a task immediately on demand, call [`task.execute()`](/task-lifecycle#execute). See [Migrating from v3](/migrating-from-v3).

## Examples

### Default options

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', () => {
  console.log('Running every minute');
});
```

Runs immediately on creation, in the system timezone, with overlapping runs allowed and no execution limit.

### Custom timezone

Evaluate the expression in a specific timezone, regardless of where the server runs:

```js
import cron from 'node-cron';

const task = cron.schedule('0 0 * * *', () => {
  console.log('Midnight in São Paulo');
}, {
  timezone: 'America/Sao_Paulo',
});
```

### Prevent overlapping runs

If a task can take longer than its interval, `noOverlap` skips a run rather than starting it on top of the previous one:

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', async () => {
  await slowJob(); // may take longer than a minute
}, {
  noOverlap: true,
});
```

A skipped run emits an [`execution:overlap`](/event-listening) event.

### Limit the number of runs

The task destroys itself after the limit is reached:

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', () => {
  console.log('This runs 5 times, then the task is destroyed');
}, {
  maxExecutions: 5,
});
```

Reaching the limit emits [`execution:maxReached`](/event-listening). For a one-shot task, set `maxExecutions: 1`.

### Add jitter to avoid a thundering herd

When many instances schedule the same job at the same instant (e.g. across a fleet), `maxRandomDelay` staggers them:

```js
import cron from 'node-cron';

const task = cron.schedule('0 * * * *', () => {
  refreshCache();
}, {
  maxRandomDelay: 30_000, // up to 30s of random delay per run
});
```

### Run on one instance across a fleet

When several copies of your app run the same schedule, `distributed` makes a task fire on only one instance per scheduled time. Out of the box you designate the runner with the `NODE_CRON_RUN` env var; for high availability, register a [run coordinator](/distributed-coordination#high-availability-a-custom-run-coordinator) (e.g. Redis):

```js
import cron from 'node-cron';

cron.schedule('0 3 * * *', runNightlyBackup, {
  name: 'nightly-backup', // required: forms the coordination key
  distributed: true,
});
```

A not-elected instance emits [`execution:skipped`](/event-listening) instead of running. See [Distributed Coordination](/distributed-coordination) for the full picture.

### Name a task

A `name` makes logs and dashboards readable, and is exposed as `task.name`:

```js
import cron from 'node-cron';

const task = cron.schedule('0 3 * * *', () => {}, {
  name: 'nightly-backup',
});

console.log(task.name); // 'nightly-backup'
```

### Tolerate late executions

A heartbeat is armed to fire at the scheduled time, but long timers drift (OS sleep, GC, CPU throttling, clock skew), so the callback can wake a little late. By default a run that wakes within `missedExecutionTolerance` (1000ms) still executes; later than that, it is reported as [missed](/event-listening). On an underpowered host, or a daily/weekly job that can wake several seconds late, raise it:

```js
import cron from 'node-cron';

const task = cron.schedule('0 3 * * 0', runWeeklyBackup, {
  missedExecutionTolerance: 5 * 60_000, // still run if we wake up to 5 min late
});
```

The tolerance is always capped to the gap until the next run, so it can never run the same slot twice. A late run fires once and the next slot is scheduled normally.

> 🛈 **Background tasks** accept two extra options, `executeTimeout` and `startTimeout`, covered in [Background Tasks](/background-tasks#manual-execution-and-executetimeout).

## Next steps

- **[Events & Observability](/event-listening)**: hook into overlap, missed runs, failures, and more.
- [Logging](/logging): route node-cron's output through your own logger.
