---
name: contao-cron-jobs-messaging
description: >-
  Run background work in a Contao extension — cron jobs (#[AsCronJob] with
  intervals), the Jobs API (create/persist/track background jobs), and
  asynchronous Symfony Messenger (message classes + #[AsMessageHandler],
  dispatching to the bus). Use when adding a cron job, a Job, or a message
  handler in a Contao bundle, or wiring Messenger transports/priorities.
---

# Contao: Cron Jobs, Jobs API & Async Messaging

Three distinct mechanisms for background work in a Contao bundle. Pick by
need:

- **Cron job** — recurring work on a schedule (cleanup, sync). Use `#[AsCronJob]`.
- **Async Messenger** — fire-and-forget work triggered from a request, run
  outside the HTTP cycle. Use a message class + `#[AsMessageHandler]`.
- **Jobs API** — persist and track a long-running, user- or system-owned task
  so its status/progress shows in the Contao back end. Usually combined with
  Messenger.

Contao is a Symfony app; extensions are Symfony bundles. App code lives under
the auto-registered `App\` namespace in `src/`, so an attribute on a class in
`src/` is enough to register it — no manual service config needed.

---

## 1. Cron jobs — `#[AsCronJob]`

Register with `Contao\CoreBundle\DependencyInjection\Attribute\AsCronJob`. Its
first constructor argument is the **interval**; an optional `method` argument
names the method to call (defaults to `__invoke`).

```php
namespace App\Cron;

use Contao\CoreBundle\DependencyInjection\Attribute\AsCronJob;

#[AsCronJob('hourly')]
class ExampleCron
{
    public function __invoke(): void
    {
        // recurring work …
    }
}
```

**Interval values** (per `cron.md`): `minutely`, `hourly`, `daily`, `weekly`,
`monthly`, `yearly`, or a full CRON expression like `*/5 * * * *`.

**Rules**

- Prefer the attribute. The legacy equivalent is the `contao.cronjob` service
  tag with `interval` (and optional `method`); annotations exist only for PHP 7
  support. All three are equivalent.
- The handler receives a `$scope` string. Compare against
  `Cron::SCOPE_WEB` / `Cron::SCOPE_CLI` (`Contao\CoreBundle\Cron\Cron`). To skip
  a run without consuming the interval, throw
  `Contao\CoreBundle\Exception\CronExecutionSkippedException` — the last-run
  time stays untouched so it retries next opportunity.
- **Web scope is time-limited** (PHP execution limit, typically 30s). Jobs that
  may run long must run CLI-only (skip in `SCOPE_WEB`).
- The `_contao/cron` web route runs only web-scoped jobs; CLI-only jobs
  (e.g. the worker supervisor) never fire from it.
- Execution is **synchronous** in tag order. For parallel/long work, return a
  `GuzzleHttp\Promise\PromiseInterface` (5.1+); for spawned processes use
  `Contao\CoreBundle\Util\ProcessUtil` (`createPromise()`,
  `createSymfonyConsoleProcess()`).

**Running**: `vendor/bin/contao-console contao:cron` (recommended via a real
minutely system crontab). Force a single job for testing:
`contao:cron "App\Cron\ExampleCron" --force`. Last runs are tracked in
`tl_cron_job`.

Scope, promises and `ProcessUtil` end-to-end → [references/cron.md](references/cron.md).

---

## 2. Async messaging — Symfony Messenger

Contao (Managed Edition) wires Messenger so async work runs even on shared
hosting without a process manager. You write a **message** + a **handler**; the
infrastructure (transports, web worker, cron-driven workers) is provided.

```php
// src/Messenger/CreateAsyncZipFileMessage.php
namespace App\Messenger;

use Symfony\Component\Messenger\Attribute\AsMessage;

#[AsMessage('contao_prio_high')] // Contao 5.7+ / Symfony 7.4+
class CreateAsyncZipFileMessage
{
    public function __construct(public array $fileIds) {}
}
```

```php
// src/Messenger/CreateAsyncZipFileMessageHandler.php
namespace App\Messenger;

use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class CreateAsyncZipFileMessageHandler
{
    public function __invoke(CreateAsyncZipFileMessage $message): void
    {
        foreach ($message->fileIds as $fileId) {
            // heavy work, runs outside the request …
        }
    }
}
```

Dispatch from a controller/service via the default message bus
(`MessageBusInterface::dispatch()`); Contao routes and processes it.

**Transports / priorities** (Contao provides three Doctrine-backed queues in
`tl_message_queue`): `contao_prio_high`, `contao_prio_normal`,
`contao_prio_low` (plus `sync` and `contao_failure`). Route a message by
naming the transport in `#[AsMessage('contao_prio_*')]` (5.7+) or via
`framework.messenger.routing` YAML.

**How it actually runs** (you do not configure this for the default queues):
- **WebWorker** consumes the configured transports during `kernel.terminate`
  (deferred after the response), but stands down when it detects a real
  `messenger:consume` worker (grace period, default 10 min).
- **Cron-driven workers**: a minutely `contao:cron` spawns time-limited
  `messenger:consume` processes (autoscaling), acting as a process manager.

To use a custom transport, add it to
`contao.messenger.web_worker.transports` and/or `contao.messenger.workers`.
Disable web-worker fallback with `web_worker.transports: []`; disable cron
workers with `workers: []` (when you run a real process manager).

Transport config, routing variants, WebWorker/worker tuning →
[references/messenger.md](references/messenger.md).

---

## 3. Jobs API (experimental, 5.7+)

> **Experimental** — not covered by Contao's BC promise; classes marked
> `@experimental` should be treated as internal and may change.

Persist and track background work so the Contao back end can show status,
progress, attachments and live updates. Entry point: the
`Contao\CoreBundle\Job\Jobs` service (inject it). A `Job`
(`Contao\CoreBundle\Job\Job`) is an **immutable DTO** — mutating methods return
a modified copy you must re-`persist()`.

```php
use Contao\CoreBundle\Job\Jobs;

public function __construct(private Jobs $jobs) {}

// current back end user, else a system job:
$job = $this->jobs->createJob('data_export');
// or: ->createSystemJob('cache_clear', public: true)
//     ->createUserJob('import_task', $userId)

$job = $job->markPending();
$this->jobs->persist($job);

// … work …
$job = $job->withProgressFromAmounts($done, $total); // or ->withProgress(42.5)
$this->jobs->persist($job);

$this->jobs->addAttachment($job, 'report.txt', "Done.\n");
$job = $job->markCompleted(); // sets progress to 100
$this->jobs->persist($job);
```

**Job model**: UUID, type (string), `Status` enum (`new`, `pending`,
`completed`, …), `Owner` (system or back end user), created-at, public flag.

**Key methods** — read: `getUuid()`, `getType()`, `getStatus()`, `getOwner()`,
`getCreatedAt()`, `isPublic()`, `isCompleted()`. Transition (return copies):
`markPending()`, `markCompleted()`, `markFailed(array $errors)`,
`markFailedBecauseRequiresCLI()`, `withProgress()`,
`withProgressFromAmounts()`, `withWarnings()`, `withErrors()`,
`withMetadata()`. Retrieve via `Jobs::findMyNewOrPending()` /
`Jobs::getByUuid($uuid)`.

Typical pattern: create the job in a request, hand its UUID to an async
message, and update progress/attachments from the `#[AsMessageHandler]`. Full
handler example → [references/jobs.md](references/jobs.md).
