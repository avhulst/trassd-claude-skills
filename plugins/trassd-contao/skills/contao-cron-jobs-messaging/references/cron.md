# Cron jobs — scope, promises, processes

Backing doc: `framework/cron.md`. Attribute, scope constants, exception,
`ProcessUtil` verified against `core-bundle/src/Cron`, `.../Exception`,
`.../Util`.

## Scope-aware cron job

A cron job receives a `$scope` string. Use it to keep long work off the
time-limited web request, and `CronExecutionSkippedException` so a skipped run
does not consume the interval (it retries at the next opportunity).

```php
namespace App\Cron;

use Contao\CoreBundle\Cron\Cron;
use Contao\CoreBundle\DependencyInjection\Attribute\AsCronJob;
use Contao\CoreBundle\Exception\CronExecutionSkippedException;

#[AsCronJob('hourly')]
class HourlyCron
{
    public function __invoke(string $scope): void
    {
        if (Cron::SCOPE_WEB === $scope) {
            // do not run in the web request — retry on the CLI
            throw new CronExecutionSkippedException();
        }

        // … CLI-only work …
    }
}
```

`Cron::SCOPE_WEB === 'web'`, `Cron::SCOPE_CLI === 'cli'`.

## Named method instead of __invoke

```php
#[AsCronJob('daily', method: 'cleanup')]
class MaintenanceCron
{
    public function cleanup(string $scope): void { /* … */ }
}
```

With a named interval and no `method`, Contao also accepts conventional method
names (e.g. `onMinutely`); `__invoke` is the default otherwise.

## Legacy service tag (equivalent to the attribute)

```yaml
# config/services.yaml
services:
    App\Cron\ExampleCron:
        tags:
            - { name: contao.cronjob, interval: hourly }
```

Only `interval` is required; add `method` if not using `__invoke`.

## Asynchronous / long-running work (5.1+)

Return a `GuzzleHttp\Promise\PromiseInterface` so the job starts in parallel
without blocking other cron jobs. For spawned processes use the
`Contao\CoreBundle\Util\ProcessUtil` service.

```php
namespace App\Cron;

use Contao\CoreBundle\Cron\Cron;
use Contao\CoreBundle\DependencyInjection\Attribute\AsCronJob;
use Contao\CoreBundle\Exception\CronExecutionSkippedException;
use Contao\CoreBundle\Util\ProcessUtil;
use GuzzleHttp\Promise\PromiseInterface;

#[AsCronJob('hourly')]
class HourlyCron
{
    public function __construct(private ProcessUtil $processUtil) {}

    public function __invoke(string $scope): PromiseInterface
    {
        if (Cron::SCOPE_WEB === $scope) {
            throw new CronExecutionSkippedException();
        }

        // Spawn a Symfony console command; ProcessUtil finds the PHP binary.
        return $this->processUtil->createPromise(
            $this->processUtil->createSymfonyConsoleProcess(
                'app:my-command', '--option-1', 'argument-1',
            ),
        );
    }
}
```

## Running & testing

```bash
# run all due cron jobs (recommended via a real minutely system crontab)
vendor/bin/contao-console contao:cron

# run a single cron job, ignoring its interval
vendor/bin/contao-console contao:cron "App\Cron\ExampleCron" --force

# list all registered cron jobs
vendor/bin/contao-console debug:container --tag contao.cronjob
```

Linux crontab entry:

```
* * * * * /usr/bin/php /path/to/contao/vendor/bin/contao-console contao:cron
```

Last-run timestamps are stored in `tl_cron_job`. The CLI has no HTTP request
context — set `router.request_context.host`/`scheme` (or a website-root domain)
if a job needs to generate URLs.
