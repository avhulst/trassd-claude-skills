# Jobs API — full patterns

Backing doc: `framework/jobs.md`. Service/DTO method names verified against
`core-bundle/src/Job` (`Jobs.php`, `Job.php`, `Status.php`, `Owner.php`,
`Attachment.php`).

> **Experimental (5.7+)** — outside Contao's BC promise; treat `@experimental`
> classes as internal and expect behavioral changes.

## The model

A `Job` (`Contao\CoreBundle\Job\Job`) is an **immutable DTO**. Every mutator
returns a new copy — you must `persist()` the returned object, not the original.

- UUID, type (string), `Status` enum, `Owner`, created-at, public flag.
- `Status` cases present in the code reference: `new`, `pending`, `completed`
  (the doc lists these with "etc.").

## Creating

```php
use Contao\CoreBundle\Job\Jobs;

public function __construct(private Jobs $jobs) {}

// current back end user if logged in, otherwise a system job
$job = $this->jobs->createJob('data_export');

// system-owned, also listed for other users
$job = $this->jobs->createSystemJob('cache_clear', public: true);

// owned by a specific user, visible only to them
$job = $this->jobs->createUserJob('import_task', $userId);
```

## Retrieving & persisting

```php
$jobs = $this->jobs->findMyNewOrPending();   // Job[] for the current user
$job  = $this->jobs->getByUuid($uuid);       // Job|null

$job = $job->markPending();
$this->jobs->persist($job);
```

## Mutators (all return a modified copy)

```php
$job = $job->markPending();
$job = $job->markCompleted();                       // also sets progress to 100
$job = $job->markFailed(['my_error']);              // also calls withErrors()
$job = $job->markFailedBecauseRequiresCLI();        // helper for CLI-only work
$job = $job->withProgress(42.5);                    // 0–100
$job = $job->withProgressFromAmounts(50, 200);      // => 25.0
$job = $job->withProgressFromAmounts(10, null);     // unknown total: log curve, caps at 95%
$job = $job->withWarnings(['my_warning']);          // translation keys
$job = $job->withErrors(['my_error']);              // translation keys
$job = $job->withMetadata(['offset' => 11]);        // serializable; your own state
```

## Attachments

Contao handles back-end downloads transparently.

```php
$this->jobs->addAttachment($job, 'report.txt', "Export finished.\nRows: 123\n");

$stream = fopen('/path/to/export.zip', 'rb');     // recommended for large files
$this->jobs->addAttachment($job, 'export.zip', $stream);
fclose($stream);
```

## End-to-end: a job driven by an async message

Create the job in a request, pass its UUID into a message, and update
progress/attachments from the handler.

```php
use Contao\CoreBundle\Job\Jobs;
use Doctrine\DBAL\Connection;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class MyMessageHandler
{
    public function __construct(
        private readonly Jobs $jobs,
        private readonly Connection $connection,
    ) {}

    public function __invoke(MyMessage $message): void
    {
        $job = $this->jobs->getByUuid($message->getJobId());

        if (!$job || $job->isCompleted()) {
            return; // gone or already done
        }

        $job = $job->markPending();
        $this->jobs->persist($job);

        foreach ($this->connection->fetchAllAssociative('SELECT * FROM foo') as $i => $item) {
            // heavy work …
            $job = $job->withProgressFromAmounts($i + 1); // unknown total
            $this->jobs->persist($job);
        }

        $this->jobs->addAttachment($job, 'report.txt', "Export finished.\n");
        $job = $job->markCompleted();
        $this->jobs->persist($job);
    }
}
```

> The doc's example reads `$message->getJobId()`. To carry a job UUID on a
> message cleanly, the core ships `JobIdAwareMessageInterface`
> (`getJobId()`/`setJobId()`) and a matching trait under
> `Messenger/Message` — useful, though `jobs.md` does not document them.
