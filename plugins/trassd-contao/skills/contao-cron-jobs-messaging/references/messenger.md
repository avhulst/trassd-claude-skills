# Async messaging — transports, routing, workers

Backing doc: `framework/async-messaging.md` (describes behavior as of 5.3.10).
`AsMessage`/`AsMessageHandler` usage and transport names verified against
`core-bundle/src/Messenger`. Assumes familiarity with the Symfony Messenger
component (messages, buses, handlers, transports, routing, `messenger:consume`).

## Default Managed-Edition transport configuration

```yaml
framework:
    messenger:
        buses:
            messenger.bus.default:
                middleware:
                    - doctrine_ping_connection
                    - doctrine_close_connection
        failure_transport: contao_failure
        transports:
            sync: sync://
            contao_failure:     doctrine://default?table_name=tl_message_queue&queue_name=failure&auto_setup=false
            contao_prio_high:   doctrine://default?table_name=tl_message_queue&queue_name=prio_high&auto_setup=false
            contao_prio_normal: doctrine://default?table_name=tl_message_queue&queue_name=prio_normal&auto_setup=false
            contao_prio_low:    doctrine://default?table_name=tl_message_queue&queue_name=prio_low&auto_setup=false
```

Messages are stored in `tl_message_queue` (Doctrine transport). The table has no
DCA; its schema is managed by Contao's `DoctrineSchemaListener` and applied on
`contao:migrate` — hence `auto_setup=false`.

## Routing a message to a priority transport

**Contao 5.7+ (Symfony 7.4+)** — `AsMessage` attribute (this is what Contao's
own `SearchIndexMessage` uses, routed to `contao_prio_low`):

```php
namespace App\Messenger;

use Symfony\Component\Messenger\Attribute\AsMessage;

#[AsMessage('contao_prio_high')]
class CreateAsyncZipFileMessage
{
    public function __construct(public array $fileIds) {}
}
```

**Or explicit YAML routing** (works on any version):

```yaml
framework:
    messenger:
        routing:
            'App\Messenger\CreateAsyncZipFileMessage': contao_prio_high
```

> NOTE (pre-5.7): `async-messaging.md` documents priority *interfaces*
> (`HighPriorityMessageInterface`, `NormalPriorityMessageInterface`,
> `LowPriorityMessageInterface` under `Contao\CoreBundle\Messenger\Message`) as
> an alternative to YAML routing. These interfaces are **not present** in the
> code reference (`core-bundle/src/Messenger/Message`) checked here — likely
> removed in the pinned version. Prefer the `#[AsMessage]` attribute on 5.7+
> and verify the interfaces exist before relying on them.

## Handler

```php
namespace App\Messenger;

use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class CreateAsyncZipFileMessageHandler
{
    public function __invoke(CreateAsyncZipFileMessage $message): void
    {
        foreach ($message->fileIds as $fileId) {
            // long-running work, off the request …
        }
    }
}
```

Dispatch from a controller/service via the default bus:

```php
use Symfony\Component\Messenger\MessageBusInterface;

public function __construct(private MessageBusInterface $bus) {}
// …
$this->bus->dispatch(new CreateAsyncZipFileMessage($fileIds));
```

## Who runs the messages

You normally configure nothing for the three default queues.

1. **WebWorker** — consumes the configured transports on `kernel.terminate`
   (after the response is sent, via `fastcgi_finish_request()`), bounded by
   `--time-limit` derived from `max_execution_time`. It detects a real running
   `messenger:consume` worker via `WorkerStartedEvent`/`WorkerRunningEvent` and
   a cache entry valid for a grace period (default 10 min); while a worker is
   seen, the WebWorker stays idle for that transport.

   ```yaml
   contao:
       messenger:
           web_worker:
               transports: [contao_prio_high, contao_prio_normal, contao_prio_low]
               grace_period: 'PT5M'   # \DateInterval duration
   ```

2. **Cron-driven workers** — with a real minutely `contao:cron`, Contao spawns
   time-limited (`--time-limit=60`) `messenger:consume` processes per transport,
   with autoscaling — effectively a process manager on shared hosting.

   ```yaml
   contao:
       messenger:
           workers:
               -
                   transports: [contao_prio_high]
                   options: ['--time-limit=60', '--sleep=5']
                   autoscale: { desired_size: 5, max: 10 }
   ```

## Custom transports & disabling fallbacks

- Custom transport: add it to `framework.messenger.transports`, then add it to
  `contao.messenger.web_worker.transports` and/or `contao.messenger.workers`.
- Real process manager instead of cron workers: `contao.messenger.workers: []`.
- No web-process execution at all: `contao.messenger.web_worker.transports: []`.

Reference real implementations: `SearchIndexMessage`,
`SearchIndexMessageHandler`, `SearchIndexListener` in core-bundle.
