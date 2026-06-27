---
name: symfony-ai-chat-memory
description: Build conversational chat with the Symfony AI Chat component and memory — managing message history, persisting conversations, agent memory, and context compression. Triggers when implementing a chatbot, conversation memory, message stores, or trimming/summarizing context for long conversations.
---

# Symfony AI: Chat & Memory

Build conversational agents with the **Chat component** (`symfony/ai-chat`),
persist and load conversation history through **message stores**, give the
agent **memory** of user facts, and keep long conversations within context
limits via **context compression**.

Install: `composer require symfony/ai-chat`.

## The Chat abstraction

`Symfony\AI\Chat\Chat` wraps an `AgentInterface` and a `MessageStoreInterface`,
handling history load/save around each turn so you don't manage the
`MessageBag` manually.

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\OpenAi\Factory;
use Symfony\AI\Platform\Message\Message;

$platform = Factory::createPlatform($apiKey);
$agent = new Agent($platform, 'gpt-4o-mini');
$chat = new Chat($agent, new InMemoryStore());

$chat->submit(Message::ofUser('Hello'));   // returns an AssistantMessage
```

`ChatInterface` exposes three methods:

- `initiate(MessageBag $messages): void` — seed the store, e.g. with the system
  message via `Message::forSystem(...)`.
- `submit(UserMessage $message): AssistantMessage` — send one user turn and get
  the reply; the user and assistant messages are persisted automatically.
- `stream(UserMessage $message): \Generator` — yield deltas in real time.

### Streaming

`stream()` yields `DeltaInterface` chunks; filter for `TextDelta` to print
text. The assistant message is persisted only **once the stream is fully
consumed**, so always drain the generator. See
[references/streaming.md](references/streaming.md).

> Streaming is **not** recommended with the Session-backed
> `Symfony\AI\Chat\Bridge\Session\MessageStore` (documented limitation).

## Message & conversation model

A conversation is a `Symfony\AI\Platform\Message\MessageBag` of messages built
via the `Message` factory:

- `Message::forSystem($text)` — instructions; keep it pinned through any
  compression.
- `Message::ofUser($text)` — a `UserMessage`.
- assistant replies arrive as `AssistantMessage`.

Read text from a message with `->asText()`. Useful `MessageBag` helpers:
`getSystemMessage()`, `withoutSystemMessage()`, `getMessages()`.

## Persisting & loading history (message stores)

A store implements `Symfony\AI\Chat\MessageStoreInterface` — just `save(MessageBag)`
and `load(): MessageBag`. The `Chat` loads before a turn and saves after, so the
chosen store decides the **scope and lifetime** of the conversation:

- **`Symfony\AI\Chat\InMemory\Store`** — single process/request only. Default
  for quick starts and tests.
- **Session** (`Bridge\Session\MessageStore`) — per-user current session.
- **Cache / Redis / Pogocache / Cloudflare** — external, shorter-lived.
- **Doctrine DBAL** (`Bridge\Doctrine\DoctrineDbalMessageStore`), MongoDB,
  Meilisearch, SurrealDb — long-term, durable history.

Pick a store by lifetime; the rest of the chat code is identical. Custom store
and managed-store (`setup`/`drop`) details, plus the `ai:message-store:setup` /
`ai:message-store:drop` console commands, are in
[references/stores.md](references/stores.md).

## Adding memory

Memory injects user facts into the **system prompt** (not the message history),
keeping context separate from the conversation. Provided by the Agent component
(`symfony/ai-agent`), wired through input processors:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

$facts = new StaticMemoryProvider(
    'My name is Wilhelm Tell',
    'I want to master the bow and arrow',
);
$memoryProcessor = new MemoryInputProcessor([$facts]);

$agent = new Agent($platform, 'gpt-4o-mini', [$systemPromptProcessor, $memoryProcessor]);
```

- `StaticMemoryProvider` — a fixed list of always-available facts.
- `EmbeddingProvider($platform, $model, $store)` — recalls facts from a large
  knowledge base by semantic similarity; pass it to `MemoryInputProcessor` the
  same way.

Best practices: keep static memory concise, use the system prompt for *behavior*
and memory for *context*, and mind that memory consumes input tokens. AI Bundle
declarative config is in [references/memory-and-bundle.md](references/memory-and-bundle.md).

## Context compression

As history grows, token cost and context limits become a problem. Compress with
a custom `Symfony\AI\Agent\InputProcessorInterface` whose `processInput(Input $input)`
rewrites the `MessageBag` before it reaches the model. Two strategies:

- **Sliding window** — drop old turns, keep the most recent N. Fast and free but
  loses context.
- **Summarization** — an extra (cheap) LLM call condenses old turns into a
  summary folded into the system message; keep the last few turns verbatim.
  Preserves context at the cost of latency.

Always gate on a `threshold` so short chats are untouched, **never drop the
system message**, and keep 4–8 recent messages uncompressed. Use a small model
for summarization. Full processor implementations and AI Bundle
`#[AsInputProcessor]` wiring are in
[references/context-compression.md](references/context-compression.md).
