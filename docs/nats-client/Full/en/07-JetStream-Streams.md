*🇬🇧 English | [🇺🇦 Українська](../uk/07-JetStream-Streams.md)*

[← Back to contents](README.md)

# 7. JetStream: Streams

A stream is a message store; covered conceptually in
[1. Introduction to NATS](01-Introduction.md#stream-where-messages-are-stored). This
chapter is about creating and managing streams through the plugin.

---

## Entering JetStream

Every JetStream operation — streams, consumers, publishing, Key-Value — is available
through a single context object, which the plugin obtains automatically as soon as the
client connects:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Get JetStream
          │
          ├─► Get Streams Manager    ──► [this chapter]
          ├─► Get Consumers Manager  ──► 9. JetStream: Consumers
          ├─► Get Publisher          ──► 8. JetStream: Publishing
          └─► Get Key-Value Store    ──► 10. JetStream: Key-Value
```

```cpp
UNatsJetStreamContext* JS = Nats->GetJetStream();
UNatsStreamManagerImpl* Streams = JS->Streams();
```

### Is JetStream available

A server can run without JetStream enabled (the plugin handles this normally — Core works
independently,
[1. Introduction to NATS](01-Introduction.md#core-vs-jetstream-two-modes-of-the-same-server)).
Check availability:

```
Get JetStream
   │
   └─► Is Available ──► (bool)
```

Or subscribe to status changes — useful when the server comes up slower than the game:

```
Get JetStream
   │
   └─► Bind Event to On Availability Changed
            │
            └─► (bAvailable) → if true, you can now create streams
```

> Right after `On Connected`, JetStream usually isn't available yet — the check happens
> asynchronously. If you need auto-created streams without manually waiting for this event,
> use `Auto Create Streams` in
> [3. Configuration](03-Configuration.md#jetstream-auto-creating-resources): the subsystem
> waits for JetStream readiness on its own.

---

## Stream configuration

`FJetStreamStreamConfig` is the struct shared by creation, updates, and auto-creation from
Project Settings.

| Field | Type | Default | Description |
|---|---|---|---|
| `Name` | `FString` | — | A unique name, e.g. `"ORDERS"`. Convention is UPPERCASE |
| `Subjects` | `TArray<FString>` | — | The subjects the stream captures, with wildcard support: `"orders.>"` |
| `Storage` | `EJetStreamStorageType` | `Memory` | `Memory` — fast, gone on server restart. `File` — survives a restart |
| `Retention` | `EJetStreamRetentionPolicy` | `Limits` | See [below](#retention-how-long-a-message-lives) |
| `Discard` | `EJetStreamDiscardPolicy` | `Old` | `Old` — evict old messages with new ones. `New` — reject new ones once the limit is reached |
| `MaxMsgs` | `int32` | `-1` (no limit) | The maximum number of messages in the stream |
| `MaxBytes` | `int32` | `-1` (no limit) | The maximum total size |
| `MaxAge` | `int32` | `0` (no limit) | The maximum message age, in seconds |
| `MaxMsgSize` | `int32` | `-1` (server limit) | The maximum size of a single message |
| `Replicas` | `int32` | `1` | Number of replicas in a cluster (requires a multi-node NATS cluster) |
| `NoAck` | `bool` | `false` | Disable publish acknowledgment (not recommended) |

> `MaxMsgs`/`MaxBytes` set to `-1` mean "no limit" — on a production server that means
> unbounded disk or memory growth. Set a sensible limit deliberately, don't leave the
> default unconsidered.

### Retention: how long a message lives

| Value | Behavior | Typical use |
|---|---|---|
| `Limits` (default) | Messages live until `MaxMsgs`/`MaxBytes`/`MaxAge` is exhausted | An event log, metrics, history |
| `Interest` | Messages are removed as soon as every active consumer has acknowledged them | Data only needed while there's at least one interested subscriber |
| `WorkQueue` | A message is removed immediately after being acknowledged by **any** consumer | A task queue: each task is processed exactly once |

### Builder (C++)

In C++ it's more convenient to build the configuration via
`FJetStreamStreamConfigBuilder` — the same result as manually filling in the struct, but
with validation and no risk of forgetting a required field:

```cpp
FJetStreamStreamConfig Config = FJetStreamStreamConfigBuilder()
    .WithName(TEXT("ORDERS"))
    .WithSubject(TEXT("orders.>"))
    .WithStorage(EJetStreamStorageType::File)
    .WithRetention(EJetStreamRetentionPolicy::WorkQueue)
    .WithMaxAge(FTimespan::FromDays(30))
    .Build();

if (TOptional<FString> Error = FJetStreamStreamConfigBuilder()
        .WithName(TEXT("ORDERS"))
        .Validate())
{
    UE_LOG(LogTemp, Error, TEXT("Invalid configuration: %s"), **Error);
}
```

`WithSubject` adds one subject to the list; for several at once, use
`WithSubjects(TArray<FString>)`. In Blueprint, the configuration is always filled directly
via `Break`/`Make Jet Stream Stream Config` — the builder is purely a C++ convenience.

---

## Operations

### Create Stream

```
Get Streams Manager
   │
   └─► Create Stream
          Config → Name     : "ORDERS"
          Config → Subjects : ["orders.>"]
          Config → Storage  : File
          │
          └─ bSuccess → StreamInfo → State → Messages, Bytes...
```

```cpp
FJetStreamStreamConfig Config;
Config.Name = TEXT("ORDERS");
Config.Subjects.Add(TEXT("orders.>"));
Config.Storage = EJetStreamStorageType::File;

Streams->CreateStream(Config, [](TJetStreamResult<FJetStreamStreamInfo> Result)
{
    if (Result.IsSuccess())
    {
        UE_LOG(LogTemp, Log, TEXT("Stream ready: %s"), *Result.Value.Config.Name);
    }
});
```

If a stream with that name and configuration already exists, the operation returns success
with the existing info rather than an error. This makes `Create Stream` safe to call on
every game start, without checking for existence first.

### Get Stream Info

```
Get Stream Info
   Stream Name : "ORDERS"
   │
   └─ bSuccess → StreamInfo
                     ├─ Config → settings
                     ├─ State  → Messages, Bytes, FirstSeq, LastSeq, NumConsumers
                     └─ Created
```

The cheapest way to check "does this stream exist" and look at current statistics — message
count, size taken up, and how many active consumers there are.

### Update Stream

```cpp
FJetStreamStreamConfig Config;
Config.Name = TEXT("ORDERS"); // must be the same name
Config.MaxMsgs = 100000;      // new limit

Streams->UpdateStream(Config, Callback);
```

You can change: `Subjects` (adding new ones), `MaxMsgs`/`MaxBytes`/`MaxAge`, `Retention`,
`Discard`. You **cannot** change after creation: `Name` and `Storage` — for that, you'll
need to delete the stream and create it again.

> **Lowering limits takes effect immediately.** If the current message count exceeds the
> new `MaxMsgs`, the excess is removed right away, following the `Discard` rule.

### Delete Stream

```
Delete Stream
   Stream Name : "OLD_EVENTS"
```

**Irreversible:** deletes every message in the stream and every consumer on it. If you only
need to clear out old messages, not destroy the stream entirely, lower `MaxAge`/`MaxMsgs`
via `Update Stream` instead.

### List Streams

```
List Streams
   │
   └─ bSuccess → Stream Names : ["ORDERS", "EVENTS", "TELEMETRY"]
```

Returns names only. Details for each require a separate `Get Stream Info` call.

---

## Example: a game events stream

```cpp
void AMyGameMode::SetupEventStream()
{
    UNatsJetStreamContext* JS = Nats->GetJetStream();

    FJetStreamStreamConfig Config = FJetStreamStreamConfigBuilder()
        .WithName(TEXT("GAME_EVENTS"))
        .WithSubject(TEXT("game.events.>"))
        .WithStorage(EJetStreamStorageType::File)
        .WithRetention(EJetStreamRetentionPolicy::Limits)
        .WithMaxAgeSeconds(7 * 24 * 3600) // one week
        .Build();

    JS->Streams()->CreateStream(Config, [](TJetStreamResult<FJetStreamStreamInfo> Result)
    {
        if (!Result.IsSuccess())
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to create stream: %s"), *Result.Error.Message);
        }
    });
}
```

Now everything published on `game.events.*` is stored for a week — regardless of whether
anyone is subscribed at that moment. The next step is reading these messages through a
consumer: [9. JetStream: Consumers](09-JetStream-Consumers.md).

---

**Next:** [8. JetStream: Publishing](08-JetStream-Publisher.md)
