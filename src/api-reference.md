---
outline: deep
title: API Reference
description: Complete reference for the node-cron module (schedule, createTask, validate, getTasks, getTask, and setLogger), with parameters, return types, and examples.
---

# API Reference

This page documents every public function exported by the `node-cron` module. For conceptual guides, follow the links into [Task Lifecycle](/task-lifecycle), [Scheduling Options](/scheduling-options), and [Events](/event-listening).

```js
import cron, {
  schedule,
  createTask,
  validate,
  validateDetailed,
  parse,
  getTasks,
  getTask,
  setLogger,
  setRunCoordinator,
  shutdown,
} from 'node-cron';
```

Both the default export (`cron.schedule(...)`) and named exports (`schedule(...)`) are available.

## `schedule(expression, func, options?)`

Creates a task and **starts it immediately**. Returns a [`ScheduledTask`](/task-lifecycle#the-scheduledtask-interface).

### Parameters

| Name         | Type                   | Description                                                                                                   |
| ------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| `expression` | `string`               | A valid [cron expression](/cron-syntax) (e.g. `"0 0 * * *"` for daily at midnight).                          |
| `func`       | `Function \| string`   | A function to run, **or** a path to a module exporting a `task` function. A path creates a [Background Task](/background-tasks). |
| `options`    | `Options` *(optional)* | Execution options. See [Scheduling Options](/scheduling-options).                                            |

### Options

```ts
type Options = {
  name?: string;                  // human-readable identifier
  timezone?: string;              // e.g. "America/New_York"
  noOverlap?: boolean;            // skip a run if the previous one is still going
  maxExecutions?: number;         // destroy the task after N runs
  maxRandomDelay?: number;        // jitter, in ms, added before each run
  logger?: Logger;                // per-task logger (not for background tasks)
  suppressMissedWarning?: boolean // silence the "missed execution" warning
  missedExecutionTolerance?: number // ms a late run may still execute (default 1000)
  distributed?: boolean;          // run on one instance per fire across a fleet
  runCoordinator?: RunCoordinator;// per-task coordinator (overrides the global one)
  distributedLease?: number;        // lease ms for lease-based coordinators (default 30000)
  unref?: boolean;                // don't keep the process alive for this task's timer
};
```

Background tasks accept two extra options, `executeTimeout` and `startTimeout`, documented in [Background Tasks](/background-tasks#manual-execution-and-executetimeout).

See [Scheduling Options](/scheduling-options) for details and examples.

### Returns

A [`ScheduledTask`](/task-lifecycle) with control methods:

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', () => {
  console.log('Running every minute');
});

task.stop();    // pause the task
task.start();   // start or resume it
task.destroy(); // remove it permanently
```

> 🛈 For **inline** tasks these methods are synchronous (`void`); for **background** tasks they return a `Promise`. See [Task Lifecycle](/task-lifecycle#inline-vs-background-sync-vs-async).

## `createTask(expression, func, options?)`

Same as `schedule`, but **does not start** the task; it's returned in the `stopped` state. Useful when you need to attach [event listeners](/event-listening) first, start conditionally, or control timing in tests.

```js
import cron from 'node-cron';

const task = cron.createTask('0 * * * *', () => {
  console.log('Running every hour');
});

task.start();
```

## `validate(expression)`

Returns `true` if the cron expression is syntactically valid, `false` otherwise.

```js
import cron from 'node-cron';

cron.validate('0 12 * * *'); // true
cron.validate('invalid');    // false
```

## `validateDetailed(expression)`

Like `validate`, but instead of a boolean returns a structured result describing **every** problem (which field, the offending value, and why). Useful for tooling, editors, and richer error messages.

```ts
type CronFieldError = { field: string; value?: string; message: string };
type DetailedValidation = {
  valid: boolean;
  fields?: ParsedFields;   // present only when valid
  errors: CronFieldError[];
};
```

```js
import { validateDetailed } from 'node-cron';

validateDetailed('0 12 * * *');
// { valid: true, errors: [], fields: { second: [0], minute: [0], hour: [12], ... } }

validateDetailed('99 12 * * 9');
// {
//   valid: false,
//   errors: [
//     { field: 'minute',    value: '99', message: '99 is a invalid expression for minute' },
//     { field: 'dayOfWeek', value: '9',  message: '9 is a invalid expression for week day' },
//   ]
// }
```

It reports **every** bad field at once (not just the first). See [Cron Syntax](/cron-syntax#validating-expressions).

## `parse(expression)`

Parses a cron expression into its decomposed, expanded fields, or **throws** an `Error` describing the first invalid field.

```ts
type ParsedFields = {
  second: number[];
  minute: number[];
  hour: number[];
  dayOfMonth: (number | string)[];
  month: number[];
  dayOfWeek: (number | string)[];
};
```

```js
import { parse } from 'node-cron';

parse('0 12 * * 1-5');
// { second: [0], minute: [0], hour: [12], dayOfMonth: [1, …, 31], month: [1, …, 12], dayOfWeek: [1, 2, 3, 4, 5] }

parse('0 99 * * *'); // throws: 99 is a invalid expression for hour
```

Use `parse` when you want the structured fields (or a thrown error); use [`validate`](#validate-expression) for a boolean and [`validateDetailed`](#validatedetailed-expression) for a non-throwing list of problems.

## `getTasks()`

Returns a `Map<string, ScheduledTask>` of every task currently registered by node-cron, keyed by task id. Tasks created with `schedule` or `createTask` are registered automatically, and a destroyed task removes itself from the registry. Useful for inspection, dashboards, or shutting everything down.

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', () => {
  console.log('Running every minute');
});

const tasks = cron.getTasks();

console.log(tasks.has(task.id));          // true
console.log(tasks.get(task.id) === task); // true
```

You can also iterate the map directly:

```js
import cron from 'node-cron';

for (const [id, task] of cron.getTasks()) {
  console.log(id, task.name, task.getStatus());
}
```

## `getTask(id)`

Returns the registered [`ScheduledTask`](/task-lifecycle) with the given id, or `undefined` when no task exists for that id.

```js
import cron from 'node-cron';

const task = cron.schedule('* * * * *', () => {});

const sameTask = cron.getTask(task.id); // === task
sameTask?.stop();
```

## `setLogger(logger)`

Replaces the global logger node-cron uses for its internal messages (such as the missed-execution warning), routing them through your own logger.

```js
import { setLogger } from 'node-cron';

setLogger({
  info:  (msg) => myLogger.info(msg),
  warn:  (msg) => myLogger.warn(msg),
  error: (msg, err) => myLogger.error(err ?? msg),
  debug: (msg) => myLogger.debug(msg),
});
```

The argument is any object implementing the `Logger` interface (`info`, `warn`, `error`, `debug`). See the [Logging guide](/logging) for the full interface, per-task loggers, winston/pino adapters, and the missed-execution warning.

## `setRunCoordinator(coordinator)`

Sets the process-wide [run coordinator](/distributed-coordination) used by tasks scheduled with `distributed: true`. This is the high-availability path: any instance can run a fire, only one wins. Without it, `distributed` tasks fall back to the `NODE_CRON_RUN` env-var default (a single designated runner).

```js
import { setRunCoordinator } from 'node-cron';
import { RedisLockCoordinator } from '@node-cron/redis-coordinator';

setRunCoordinator(new RedisLockCoordinator(redis));
```

The argument is any object implementing the `RunCoordinator` interface:

```ts
interface RunCoordinator {
  shouldRun(key: string, ttlMs: number): boolean | Promise<boolean>;
  onComplete?(key: string): void | Promise<void>;
}
```

Pass `undefined` to clear it. A per-task coordinator can be set via the [`runCoordinator`](/scheduling-options) option. See [Distributed Coordination](/distributed-coordination) for the full guide.

## `shutdown(timeout?)`

Gracefully tears down **every** registered task: stops each one, waits for any in-flight run to settle (up to `timeout`), then destroys them all and clears the registry. Returns a `Promise` that resolves once teardown is complete. Call it from a signal handler so a redeploy or Ctrl-C doesn't cut a running job in half.

| Name      | Type     | Default | Description                                                        |
| --------- | -------- | ------- | ------------------------------------------------------------------ |
| `timeout` | `number` | `5000`  | Max time (ms) to wait for busy tasks to finish before destroying them. Tasks that are idle don't wait. |

```js
import { shutdown } from 'node-cron';

async function stop() {
  await shutdown(10_000); // give running jobs up to 10s to finish
  process.exit(0);
}

process.once('SIGTERM', stop);
process.once('SIGINT', stop);
```

- Only tasks that are actually running when `shutdown` is called are awaited; the timeout is a ceiling, not a fixed delay.
- After it resolves, every task is `destroyed` and [`getTasks()`](#gettasks) is empty. Create new tasks if the process keeps running.
- Errors thrown while stopping or destroying an individual task are routed to the [logger](/logging), not rethrown, so one bad task can't abort the whole shutdown.

## Next steps

- [Task Lifecycle & Status](/task-lifecycle): what the returned task can do.
- [Migrating from v3](/migrating-from-v3): if you're upgrading an existing project.
