# Notify (`symfony/ux-notify`)

Sends server-pushed **native browser notifications** using Mercure. Requires
StimulusBundle configured, a **running Mercure hub**, and a configured notifier
chatter transport.

## Configure a Mercure chatter transport

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        chatter_transports:
            myMercureChatter: '%env(MERCURE_DSN)%'
```

## Send a message from PHP

Inject `ChatterInterface` and send a `ChatMessage` with `MercureOptions` naming
the topic(s) it goes to:

```php
use Symfony\Component\Notifier\ChatterInterface;
use Symfony\Component\Notifier\Message\ChatMessage;
use Symfony\Component\Notifier\Bridge\Mercure\MercureOptions;

public function __construct(private ChatterInterface $chatter) {}

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $message = (new ChatMessage(
        'Flash sales has been started!',
        new MercureOptions(['/chat/flash-sales'])
    ))->transport('myMercureChatter');

    $this->chatter->send($message);

    return 0;
}
```

## Listen in Twig

Call `stream_notifications()` with the topic(s) to subscribe to, anywhere on the
page. When a message arrives on a subscribed topic, a native browser
notification fires.

```twig
{{ stream_notifications(['/chat/flash-sales']) }}
```

Called without arguments it defaults to the `https://symfony.com/notifier` topic.

## Notes

- Choose a non-default Mercure hub with the `mercure_hub` option in
  `config/packages/notify.yaml` (defaults to `mercure.hub.default`).
- Extend behavior by passing `{'data-controller': 'mynotify'}` to
  `stream_notifications()` and listening for `notify:connect`
  (`event.detail.eventSources`).
