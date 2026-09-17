*🇬🇧 English | [🇺🇦 Українська](../uk/10-JetStream-KeyValue.md)*

[← Back to contents](README.md)

# 10. JetStream: Key-Value

Key-Value (KV) is a "key → value" store built on top of JetStream; covered conceptually in
[1. Introduction to NATS](01-Introduction.md#key-value-a-store-built-on-top-of-a-stream).
A key in KV is, in essence, the subject `$KV.<bucket>.<key>` in a hidden stream, so the
character requirements for a key are the same as for a subject: a dot separates a key's
"levels" (`player.42.stats` is a valid key with dots), and a key itself may not contain
spaces.

---

## Buckets

### CreateBucket / DeleteBucket

```
Get JetStream
   │
   └─► Get Key-Value Store
          │
          └─► Create KV Bucket
                 Config → Bucket  : "PLAYER_STATS"
                 Config → History : 1
                 Config → Storage : File
```

```cpp
UNatsKVStoreImpl* KV = Nats->GetJetStream()->KeyValue();

FJetStreamKVConfig Config = FJetStreamKVConfigBuilder()
    .WithBucket(TEXT("PLAYER_STATS"))
    .WithStorage(EJetStreamStorageType::File)
    .WithHistory(5)
    .Build();

KV->CreateBucket(Config, [](TJetStreamResult<bool> Result) {});
```

Like `Create Stream`, calling this again for an existing bucket returns success rather than
an error — safe to call on every game start.

### Bucket configuration

`FJetStreamKVConfig`:

| Field | Type | Default | Description |
|---|---|---|---|
| `Bucket` | `FString` | — | The bucket's name, e.g. `"PLAYER_STATS"` |
| `MaxAge` | `int32` | `0` (no limit) | The entry's lifetime, in seconds |
| `MaxValueSize` | `int32` | `-1` (server limit) | The maximum value size, in bytes |
| `History` | `int32` | `1` | How many versions to keep per key. `1` — the current value only |
| `Storage` | `EJetStreamStorageType` | `Memory` | `Memory` or `File` — the same principle as with streams |
| `Replicas` | `int32` | `1` | Cluster replication |

```
Delete KV Bucket
   Bucket : "OLD_BUCKET"
```

**Irreversible:** deletes the bucket and every key in it.

---

## Basic operations

### KV Put / KV Get

```
KV Put
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   Value  : "{\"level\":5,\"xp\":1200}"
   │
   └─ bSuccess → Entry → Revision : 1
```

```
KV Get
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   └─ bSuccess → Entry → Value : "{\"level\":5,\"xp\":1200}"
   └─ !bSuccess → the key doesn't exist (Error Code = Key Not Found)
```

```cpp
KV->Put(TEXT("PLAYER_STATS"), TEXT("player_42"), TEXT("{\"level\":5,\"xp\":1200}"),
    [](TJetStreamResult<FJetStreamKVEntry> Result) {});

KV->Get(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [](TJetStreamResult<FJetStreamKVEntry> Result)
    {
        if (Result.IsSuccess())
        {
            UE_LOG(LogTemp, Log, TEXT("Level: %s, revision: %d"), *Result.Value.Value, Result.Value.Revision);
        }
    });
```

### KV Delete

```
KV Delete
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
```

> **A known quirk of the current version.** If a bucket's `History` is greater than 1,
> `KV Get` after `KV Delete` may return the key's **previous** value instead of a "key not
> found" error — details in
> [11. Errors and Diagnostics](11-Errors-And-Diagnostics.md#kv-delete-in-a-bucket-with-history).
> For buckets with `History = 1` (the default), behavior is correct.

### KV List Keys

```
KV List Keys
   Bucket : "PLAYER_STATS"
   │
   └─ bSuccess → Keys : ["player_1", "player_42", "player_99"]
```

---

## Optimistic concurrency

### KV Put With Revision

Write a value only if no one else has changed the key between your read and your write —
the classic "read-modify-write" without races:

```
KV Get (Bucket: "PLAYER_STATS", Key: "player_42")
   │
   └─► Entry → Revision  ───────────┐
                                     │
[changed the data locally]          │
                                     ▼
KV Put With Revision
   Bucket             : "PLAYER_STATS"
   Key                : "player_42"
   Value              : (updated data)
   Expected Revision  : (Revision from Get above)
   │
   └─ !bSuccess → someone else wrote first — read again and retry
```

```cpp
KV->Get(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [this](TJetStreamResult<FJetStreamKVEntry> GetResult)
    {
        if (!GetResult.IsSuccess()) return;

        const FString NewValue = ApplyChange(GetResult.Value.Value);

        Nats->GetJetStream()->KeyValue()->PutWithRevision(
            TEXT("PLAYER_STATS"), TEXT("player_42"), NewValue, GetResult.Value.Revision,
            [](TJetStreamResult<FJetStreamKVEntry> PutResult)
            {
                if (!PutResult.IsSuccess())
                {
                    // A version conflict — someone wrote between Get and Put. Retry the cycle.
                }
            });
    });
```

### KV Create

"Write only if the key doesn't already exist" — an atomic operation, useful for a
distributed lock or "the first player gets the prize":

```
KV Create
   Bucket : "LOCKS"
   Key    : "boss_fight_1"
   Value  : "owned_by_session_abc"
   │
   ├─ bSuccess = true  → the key was just created by you — you're the owner
   └─ bSuccess = false → the key already existed — someone else owns it
```

```cpp
KV->Create(TEXT("LOCKS"), TEXT("boss_fight_1"), SessionId,
    [](TJetStreamResult<FJetStreamKVEntry> Result)
    {
        if (Result.IsSuccess())
        {
            StartBossFight(); // we're first
        }
        else
        {
            // Someone else already started the fight with this boss
        }
    });
```

Unlike "`KV Get` first, then `KV Put` if not found," `KV Create` is atomic: even if two
players call it at the exact same time, exactly one of them succeeds.

---

## Watch: reacting to changes in real time

### KV Watch / KV Stop Watch

```
KV Watch
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   Callback ──► (Entry) → update the UI whenever the value changes
```

```cpp
FString WatchId = KV->Watch(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [](const FJetStreamKVEntry& Entry)
    {
        UpdateStatsWidget(Entry.Value);
    });

// when no longer needed (e.g. the widget closes)
KV->StopWatch(WatchId);
```

**You can leave `Key` empty** to get changes for **every** key in the bucket — useful for
logging or full-state synchronization:

```cpp
KV->Watch(TEXT("PLAYER_STATS"), FString(), // empty Key = every key
    [](const FJetStreamKVEntry& Entry)
    {
        UE_LOG(LogTemp, Log, TEXT("Key %s changed: %s"), *Entry.Key, *Entry.Value);
    });
```

`Key` also supports the same `*`/`>` patterns as a Core subscription's subject
([1. Introduction to NATS](01-Introduction.md#subject-the-messages-address)) — for example
`"player_*.status"` for "any player's status."

> Every `KV Watch` call always returns the key's **current** value on a change, fetched with
> a separate server request — not raw bytes from the internal message. This means a small
> delay between the physical change and the callback firing, usually unnoticeable.

---

## History

```
KV History
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   └─ bSuccess → History : [Entry(rev=1), Entry(rev=2), Entry(rev=3)]  ← oldest to newest
```

Only works if the bucket was created with `History` greater than 1 — otherwise only the
current entry is returned.

---

## Binary values

The same "a Bytes variant alongside the text one" principle as elsewhere in the plugin
([6. Binary Data](06-Binary-Data.md)):

| Text | Bytes |
|---|---|
| `KV Put` | `KV Put Bytes` |
| `KV Put With Revision` | `KV Put Bytes With Revision` |
| `KV Create` | `KV Create Bytes` |

```cpp
TArray<uint8> IconBytes = ExportIconToBytes();
KV->PutBytes(TEXT("PLAYER_AVATARS"), TEXT("player_42"), IconBytes, [](auto) {});
```

A value read back (`KV Get`, `KV Watch`, `KV History`) always contains **both**
representations at once — `Value` (text) and `ValueBytes` (exact bytes), regardless of which
method wrote it:

```
KV Get
   │
   └─ bSuccess → Entry
                    ├─ Value       (text representation)
                    └─ Value Bytes (exact bytes — this is the field for binary data)
```

---

## Example: hot-reloading configuration

A typical scenario where KV replaces a separate config file that would otherwise require a
restart:

```cpp
void AGameConfigManager::BeginPlay()
{
    Super::BeginPlay();

    UNatsKVStoreImpl* KV = Nats->GetJetStream()->KeyValue();

    // Read the current value immediately at startup
    KV->Get(TEXT("CONFIG"), TEXT("difficulty_multiplier"),
        [this](TJetStreamResult<FJetStreamKVEntry> Result)
        {
            if (Result.IsSuccess())
            {
                ApplyDifficulty(FCString::Atof(*Result.Value.Value));
            }
        });

    // And react to any change from then on, without restarting the game
    KV->Watch(TEXT("CONFIG"), TEXT("difficulty_multiplier"),
        [this](const FJetStreamKVEntry& Entry)
        {
            ApplyDifficulty(FCString::Atof(*Entry.Value));
        });
}
```

A designer changes the value with `nats kv put CONFIG difficulty_multiplier 1.5` (or their
own admin panel that calls `KV Put` under the hood) — every running server picks up the
change within a second, with no release and no restart.

---

**Next:** [11. Errors and Diagnostics](11-Errors-And-Diagnostics.md)
