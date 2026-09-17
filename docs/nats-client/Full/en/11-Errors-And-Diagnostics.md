*🇬🇧 English | [🇺🇦 Українська](../uk/11-Errors-And-Diagnostics.md)*

[← Back to contents](README.md)

# 11. Errors and Diagnostics

Core and JetStream report errors differently — Core with plain text, JetStream with a
structured code. Both are covered below, along with common causes and how to look at what's
actually happening on the wire.

---

## Core: the On Error event

Core has no separate error code — just an event with a text description:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On Error
            │
            └─► (Error: FString) → Print String / log it
```

Fires for errors at any level: failing to establish a TCP connection, the server rejecting
authentication, a protocol violation. You have to tell the cause apart by the message text —
see [the table below](#common-core-messages) for typical wording.

Besides the event, every call (`Publish`, `Subscribe`, `Request Async`, etc.) returns a
`bool`/`bSuccess` — check it immediately rather than relying solely on `On Error`, which
describes general connection failures, not each individual unsuccessful call.

---

## JetStream: structured results

Every JetStream operation returns one of two things.

### C++: TJetStreamResult&lt;T&gt;

```cpp
Streams->GetStreamInfo(TEXT("ORDERS"), [](TJetStreamResult<FJetStreamStreamInfo> Result)
{
    if (Result.IsSuccess())
    {
        const FJetStreamStreamInfo& Info = Result.Value;
        // ...
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("[%s] %s"), *Result.Error.GetCodeString(), *Result.Error.Message);
    }
});
```

### Blueprint: a triple delegate

The same result in Blueprint, as three callback pins instead of one combined type:

```
Get Stream Info
   │
   └─ Callback ──► (bSuccess, StreamInfo, Error)
                       │
                       ├─ bSuccess = true  → StreamInfo is filled, Error is empty (Code = None)
                       └─ bSuccess = false → StreamInfo is default/empty, Error is filled
```

### FJetStreamError

| Field | Description |
|---|---|
| `Code` | The machine-readable code — `EJetStreamErrorCode`, [table below](#jetstream-error-codes) |
| `Message` | A human-readable description, suitable for logs |
| `Details` | Extra context (the raw server response, NATS codes) |

Helper pure nodes in the plugin's library:

| Node | What it does |
|---|---|
| **Is Success (JetStreamError)** | `Code == None` |
| **Is Error (JetStreamError)** | The opposite of `Is Success` |
| **Get Error Code String** | `Code` as text, for logs |
| **To String (JetStreamError)** | The whole field as one string — handy for `Print String` |

### JetStream error codes

| `EJetStreamErrorCode` | What happened |
|---|---|
| `None` | Success — a distinct code, not the absence of an error |
| `NotConnected` | The NATS client isn't connected |
| `JetStreamUnavailable` | JetStream isn't enabled on the server (or availability hasn't been confirmed yet — see [7. JetStream: Streams](07-JetStream-Streams.md#is-jetstream-available)) |
| `InvalidParameter` | An invalid argument — e.g. an empty stream name |
| `StreamNotFound` | No stream with that name |
| `ConsumerNotFound` | No consumer with that name |
| `StreamAlreadyExists` | The stream name is taken by a different configuration |
| `ConsumerAlreadyExists` | The consumer name is taken by a different configuration |
| `Timeout` | The server didn't respond within the allotted time |
| `PermissionDenied` | The credentials are valid, but the action isn't allowed by server policy |
| `ServerError` | The server returned an error that doesn't fit any of the codes above |
| `SerializationError` | Failed to parse the server's response as JSON |
| `BucketNotFound` | No KV bucket with that name |
| `KeyNotFound` | No such key in the bucket |
| `Unknown` | An unforeseen situation |

---

## Common Core messages

| Symptom | Cause |
|---|---|
| `Failed to connect to NATS server. Reason: ...` in `On Error` | The server is unreachable at the given address/port — a firewall, the wrong port, or the server isn't running |
| `NATS Client is already connecting` / `already connected` | A repeated `Connect` while a previous connection is still active — call `Disconnect` first |
| `Cannot publish with headers: Server does not support headers` | The server is older than NATS 2.2.0 — upgrade the server or don't use headers |
| `Publish`/`Subscribe` return `false` with nothing logged to `On Error` | The client isn't connected at the time of the call — check `Is Connected` |
| `Request Async` always ends with `"Timeout"` | The responder isn't subscribed to the right subject, or isn't publishing to `Message.ReplyTo` — see [5. Core Messaging](05-Core-Messaging.md#requestreply) |

---

## Common JetStream situations

### `JetStreamUnavailable` right after connecting

Normal: the JetStream availability check is asynchronous and takes some time after
`On Connected`. Wait for `On Availability Changed` or check `Is Available` before your
first call —
[7. JetStream: Streams](07-JetStream-Streams.md#is-jetstream-available).

### `StreamAlreadyExists` on `Create Stream`

Unlike an error, `Create Stream` with the **same** parameters for an existing stream
returns success — the `StreamAlreadyExists` code only appears when you try to create a
stream with the same name but a **different** configuration (e.g. a different `Storage`).
Rename the stream, or `Delete Stream` first.

### KV Delete in a bucket with history

If a bucket was created with `History` greater than 1, the current version of the plugin
physically removes only the key's most recent revision on `KV Delete` — and `KV Get`
afterward may return the **previous** value instead of a `KeyNotFound` error, while
`KV History` shows one fewer entry than expected. This is a known limitation, not
unpredictable behavior.

**What to do for now:** for buckets where "deleted for good" semantics matter, keep
`History = 1` (the default) — there, `KV Delete` behaves correctly. If you need history for
a particular bucket, treat `KV Delete` as "remove from active use" (for example, check a
separate flag field in the value) rather than relying on the key physically disappearing.

### Pull Messages returns an empty array

Not an error — it means no new message showed up within `Timeout Seconds`. Check
`Result.IsSuccess()`, not just whether the array is non-empty.

---

## Logs

The plugin writes to a separate category for each layer — enable only what you're currently
debugging:

| Category | What it logs |
|---|---|
| `LogNatsClient` | The core: connecting, publishing, protocol parsing |
| `LogNatsClientSubsystem` | The subsystem: connecting, reconnecting, routing JetStream messages |
| `LogNatsClientComponent` | The same for `NATS Client Component` |
| `LogNatsProtocol` | Parsing and building protocol frames — the lowest level |
| `LogNatsJetStreamContext` | JetStream initialization, availability checks |
| `LogJetStreamRequestHandler` | Requests to the server's JetStream API |
| `LogNatsStreamManager` | Stream operations |
| `LogNatsConsumerManager` | Consumer operations, push/pull delivery |
| `LogNatsJetStreamPublisher` | Publishing into JetStream |
| `LogNatsKVStore` | Key-Value operations |
| `LogJetStreamSerializer` | Serializing/deserializing JSON requests to the server |

Turn on verbose logging with a console command in the editor or game:

```
Log LogNatsClient Verbose
Log LogNatsConsumerManager VeryVerbose
```

Or permanently, in `DefaultEngine.ini`:

```ini
[Core.Log]
LogNatsClient=Verbose
LogNatsConsumerManager=VeryVerbose
```

---

## Test buttons

**Project Settings → Plugins → NATS Messaging Client**:

| Button | What it checks |
|---|---|
| **Test Connection** | Whether the server is reachable with the current settings |
| **Test JetStream** | The same, plus a request to the JetStream API — confirms JetStream is enabled |

The result is an instant popup notification in the editor, no need to launch the game.

---

## When nothing else helps

1. Enable `Log LogNatsClient VeryVerbose` (and, if this concerns JetStream,
   `LogJetStreamRequestHandler VeryVerbose`) and reproduce the issue once.
2. Check the server's monitoring page — `http://<address>:8222/varz` shows a list of active
   connections and basic stats, `http://<address>:8222/jsz` shows JetStream state.
3. If you're unsure exactly what's on the wire — `nats sub "your.subject.>"` via the
   [NATS CLI](https://github.com/nats-io/natscli) in a separate terminal shows raw messages
   independently of the plugin: if they show up there but the plugin stays silent, the
   problem is on the client side; if they don't show up there either, the problem is in the
   publish itself or on the server.

---

**Next:** [12. Testing and the Local Server](12-Testing-And-Local-Server.md)
