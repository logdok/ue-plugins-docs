*🇬🇧 English | [🇺🇦 Українська](../uk/09-JetStream-Consumers.md)*

[← Back to contents](README.md)

# 9. JetStream: Consumers

A stream stores messages; a consumer is the cursor used to read them. Covered
conceptually in
[1. Introduction to NATS](01-Introduction.md#consumer-a-read-cursor-into-a-stream). This
chapter is practical: how to create a consumer, choose push or pull, and acknowledge
processed messages.

---

## Push vs. pull: which to choose

| | **Push consumer** | **Pull consumer** |
|---|---|---|
| Who initiates delivery | The server sends it itself, as soon as a message appears | You ask yourself: "give me up to N messages" |
| Reaction speed | Instant | Depends on how often you ask |
| Control over processing pace | Limited | Full — request exactly as much as you're ready to process |
| Typical use | Real-time event handling: notifications, live state updates | Batch processing, task queues, workers that process several items at once |

The choice comes down to one configuration field — `DeliverSubject`: filled in means a push
consumer, empty means pull.

---

## Consumer configuration

`FJetStreamConsumerConfig`:

| Field | Type | Default | Description |
|---|---|---|---|
| `Name` | `FString` | — | The consumer's name, e.g. `"ORDERS_PROCESSOR"`. An empty name creates an ephemeral (temporary) consumer |
| `DeliverPolicy` | `EJetStreamDeliverPolicy` | `All` | Where to start reading from — [below](#deliverpolicy-where-to-start-reading) |
| `AckPolicy` | `EJetStreamAckPolicy` | `Explicit` | How to acknowledge — [below](#ackpolicy-how-to-acknowledge) |
| `AckWait` | `float` | `30.0` | How long to wait for an acknowledgment (seconds) before redelivering the message |
| `MaxDeliver` | `int32` | `-1` (no limit) | How many times to attempt delivering one message |
| `ReplayPolicy` | `EJetStreamReplayPolicy` | `Instant` | `Instant` — as fast as possible. `Original` — with the same spacing as the original publications |
| `FilterSubject` | `FString` | empty | Only receive messages with this subject (for streams with multiple subjects) |
| `OptStartSeq` | `int32` | `0` | The starting sequence (only for `DeliverPolicy = ByStartSequence`) |
| `OptStartTime` | `int32` | `0` | The starting time, a Unix timestamp (only for `DeliverPolicy = ByStartTime`) |
| `DeliverSubject` | `FString` | empty | Filled in → a push consumer. Empty → a pull consumer |

### DeliverPolicy: where to start reading

| Value | Behavior |
|---|---|
| `All` | From the very first message in the stream — a full history replay |
| `New` | Only messages published **after** the consumer was created |
| `Last` | Start from the most recent message |
| `LastPerSubject` | The last message for **each** subject in a stream with multiple subjects — useful for "current state" |
| `ByStartSequence` | From a specific sequence number (`OptStartSeq`) |
| `ByStartTime` | From a specific point in time (`OptStartTime`) |

### AckPolicy: how to acknowledge

| Value | Behavior |
|---|---|
| `Explicit` (default, recommended) | Each message is acknowledged individually, in any order |
| `All` | Acknowledging message #N automatically acknowledges every prior one too — strictly in order only |
| `None` | No acknowledgment needed: the message is considered delivered immediately. Risk of loss if the handler fails |

### Builder (C++)

```cpp
FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_PROCESSOR"))
    .WithDeliverPolicy(EJetStreamDeliverPolicy::New)
    .WithAckPolicy(EJetStreamAckPolicy::Explicit)
    .WithAckWaitSeconds(60.0f)
    .WithFilterSubject(TEXT("orders.created"))
    .Build();
```

For a push consumer — `AsPushConsumer()` (generates a unique `DeliverSubject` itself if you
haven't set your own) or `WithDeliverSubject(TEXT("..."))` explicitly.

---

## Push consumer

### Create Consumer + Subscribe To Consumer

```
Get Consumers Manager
   │
   ├─► Create Consumer
   │      Stream Name          : "ORDERS"
   │      Config → Name        : "ORDERS_LIVE"
   │      Config → Deliver Subject : "_INBOX.orders_live"   ← any unique string
   │      │
   │      └─ bSuccess ──►
   │
   └─► Subscribe To Consumer
          Stream Name    : "ORDERS"
          Consumer Name  : "ORDERS_LIVE"
```

After `Subscribe To Consumer`, messages arrive as an event:

```
Get Consumers Manager
   │
   └─► Bind Event to On Message Received
            │
            └─► (Message: FJetStreamMessage)
                    │
                    ├─► Break Jet Stream Message
                    │       ├─ Nats Msg → Data     (the actual content)
                    │       ├─ Sequence
                    │       └─ Num Delivered        (>1 means a redelivery)
                    │
                    └─► Ack Message (Message)   ← required after successful processing!
```

```cpp
UNatsConsumerManagerImpl* Consumers = Nats->GetJetStream()->Consumers();

FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_LIVE"))
    .AsPushConsumer()
    .Build();

Consumers->CreateConsumer(TEXT("ORDERS"), Config,
    [Consumers](TJetStreamResult<FJetStreamConsumerInfo> Result)
    {
        if (Result.IsSuccess())
        {
            Consumers->Subscribe(TEXT("ORDERS"), Result.Value.Config.Name,
                [](TJetStreamResult<FJetStreamConsumerInfo>) {});
        }
    });

Consumers->OnMessageReceived.AddDynamic(this, &AMyActor::HandleOrder);

// ...
void AMyActor::HandleOrder(const FJetStreamMessage& Message)
{
    ProcessOrder(Message.NatsMsg.Data);
    Nats->GetJetStream()->Consumers()->AckMessage(Message);
}
```

> **There's one event for all push consumers.** If you're subscribed to several consumers at
> once, tell messages apart using `Message.Consumer` or `Message.NatsMsg.Subject`.

---

## Pull consumer

A configuration without `DeliverSubject`, and instead of subscribing, an explicit request
for the desired number of messages:

```
Get Consumers Manager
   │
   ├─► Create Consumer
   │      Stream Name    : "ORDERS"
   │      Config → Name  : "ORDERS_BATCH"
   │      (Deliver Subject stays empty)
   │
   └─► Pull Messages
          Stream Name     : "ORDERS"
          Consumer Name   : "ORDERS_BATCH"
          Batch Size      : 10
          Timeout Seconds : 5.0
          │
          └─ bSuccess → Messages : (an array of up to 10 elements)
                            │
                            └─► For Each Loop
                                    └─► Ack Message
```

```cpp
FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_BATCH"))
    .Build(); // DeliverSubject empty by default — pull

Consumers->CreateConsumer(TEXT("ORDERS"), Config, [](auto) {});

Consumers->PullMessages(TEXT("ORDERS"), TEXT("ORDERS_BATCH"), 10, 5.0f,
    [Consumers](TJetStreamResult<TArray<FJetStreamMessage>> Result)
    {
        if (Result.IsSuccess())
        {
            for (const FJetStreamMessage& Message : Result.Value)
            {
                ProcessOrder(Message.NatsMsg.Data);
                Consumers->AckMessage(Message);
            }
        }
    });
```

If nothing shows up within `Timeout Seconds`, `Messages` comes back as an empty array with
`bSuccess = true`: an empty queue is a normal result, not an error. If there are fewer
messages than `Batch Size`, you get however many there are.

---

## Acknowledgment: Ack / Nak / Term

| Node | Meaning | Consequence |
|---|---|---|
| **Ack Message** | "Processed successfully" | Won't be redelivered. For a `WorkQueue` stream, it's removed from the stream |
| **Nak Message** | "Failed, try again" | Redelivered after a short delay; `NumDelivered` increases |
| **Term Message** | "Don't try again" | The message will NEVER be redelivered, even without successful processing |

```
[after processing an order]
   │
   ├─ success                       → Ack Message
   ├─ temporary error (DB unreachable) → Nak Message
   └─ corrupted/invalid data        → Term Message
```

**The key rule:** if `AckWait` elapses without an acknowledgment, the server considers the
message unprocessed and delivers it again. This applies to both push and pull consumers
alike. If processing is systematically slower than the current `AckWait`, increase
`AckWait` in the configuration rather than trying to acknowledge "just in case" right after
receiving.

---

## Get Consumer Info / Delete Consumer

```
Get Consumer Info
   Stream Name   : "ORDERS"
   Consumer Name : "ORDERS_LIVE"
```

Useful to check whether a consumer exists before subscribing to it or pulling messages from
it, rather than assuming it's already been created.

```
Delete Consumer
   Stream Name   : "ORDERS"
   Consumer Name : "OLD_PROCESSOR"
```

Only deletes the consumer and its read position — messages in the stream itself are
untouched. Often used as "reset progress": delete and recreate with a different
`DeliverPolicy`.

---

## Example: a task queue (work queue)

A complete working cycle combining a stream, the `WorkQueue` retention policy, and a pull
consumer — a classic task queue where each task is processed by exactly one worker:

```cpp
// One-time setup
FJetStreamStreamConfig StreamConfig = FJetStreamStreamConfigBuilder()
    .WithName(TEXT("TASKS"))
    .WithSubject(TEXT("tasks.pending"))
    .WithRetention(EJetStreamRetentionPolicy::WorkQueue) // removed immediately after Ack
    .Build();
Streams->CreateStream(StreamConfig, [](auto) {});

FJetStreamConsumerConfig ConsumerConfig = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("WORKER"))
    .WithAckPolicy(EJetStreamAckPolicy::Explicit)
    .WithAckWaitSeconds(120.0f) // enough for long processing
    .Build();
Consumers->CreateConsumer(TEXT("TASKS"), ConsumerConfig, [](auto) {});

// Enqueueing a task (from anywhere in the game)
Publisher->Publish(TEXT("tasks.pending"), TaskJson, [](auto) {});

// The worker's loop (e.g. every few seconds, or right after the previous batch finishes)
void AWorker::PullNextBatch()
{
    Consumers->PullMessages(TEXT("TASKS"), TEXT("WORKER"), 5, 10.0f,
        [this](TJetStreamResult<TArray<FJetStreamMessage>> Result)
        {
            for (const FJetStreamMessage& Task : Result.Value)
            {
                const bool bOk = ProcessTask(Task.NatsMsg.Data);
                if (bOk)
                {
                    Consumers->AckMessage(Task);
                }
                else
                {
                    Consumers->NakMessage(Task); // try again later
                }
            }
            PullNextBatch(); // the next batch
        });
}
```

If several workers pull from the same pull consumer at once, each message goes to only one
of them: JetStream guarantees the same message is never handed out to two concurrent
`Pull Messages` requests.

---

**Next:** [10. JetStream: Key-Value](10-JetStream-KeyValue.md)
