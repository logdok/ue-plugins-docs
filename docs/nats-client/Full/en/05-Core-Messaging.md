*🇬🇧 English | [🇺🇦 Українська](../uk/05-Core-Messaging.md)*

[← Back to contents](README.md)

# 5. Core Messaging

This chapter covers Core mode: connecting, publishing, subscribing and request/reply. If
the terms "subject," "wildcard" or "request/reply" aren't familiar yet, read
[1. Introduction to NATS](01-Introduction.md) first.

---

## Three ways to use the plugin

All three give you the same set of operations (`Publish`, `Subscribe`, `Request Async`,
etc.) — the difference is only in who manages the connection's lifetime.

| | **NATS Client Subsystem** | **NATS Client Component** | **NATS Core Client** (`UNatsClient`) |
|---|---|---|---|
| What it is | A `GameInstanceSubsystem` — one instance for the whole game | An `ActorComponent` — one per actor | A raw connection object |
| Lives | The whole game, survives level changes | As long as the owning actor is alive | As long as you hold onto it |
| Gets settings from | Project Settings | The component's own properties | Parameters you pass yourself |
| When to choose | **The default choice.** One shared message connection for the whole game | The connection logically belongs to a specific actor (for example, a standalone NPC service) | Advanced scenarios: manual lifecycle control, several independent connections at once |

**The default recommendation is the subsystem.** The rest of this chapter's examples use it;
for the component and the raw client, the function signatures are identical.

> **Auto-reconnect only exists in the subsystem and the component.** The raw `NATS Core
> Client` does nothing on its own after a disconnect — it just reports it with an
> `On Disconnected` event and waits for your next `Connect` call. If you choose the raw
> client for full control, you'll need to implement retry logic yourself.

```
Get Game Instance Subsystem (Nats Client Subsystem)
```

— one node, available from any graph. You don't need to create the object or store it in a
variable: the subsystem already exists by the time the game starts and returns the same
instance every time.

---

## Connecting

### Connect To Server

Connects using the values from [Project Settings](03-Configuration.md) — the address, port
and credentials. The simplest option when there's one configuration for the whole project.

```
Event BeginPlay
   │
   └─► Get Game Instance Subsystem (Nats Client Subsystem)
          │
          └─► Connect To Server
```

### Connect

Connects with an explicit address and port, bypassing Project Settings:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Connect
          Server URL     : "192.168.1.100"
          Port           : 4222
          b Non Blocking : false
```

`b Non Blocking` (default `false`) only affects how the socket is created; it doesn't
affect the speed of the connection itself — `On Connected` always fires asynchronously,
once the TCP and NATS handshakes finish.

### Connect With Credentials

The same, but with explicit credentials — the typical choice when the data arrives at
runtime (detailed, with an example, in
[3. Configuration](03-Configuration.md#where-to-set-credentials)):

```cpp
UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();

FNatsCredentials Credentials;
Credentials.AuthType = ENatsAuthType::Basic;
Credentials.Username = TEXT("player123");
Credentials.Password = TEXT("secret");

Nats->ConnectWithCredentials(TEXT("nats.example.com"), 4222, Credentials);
```

### Calling Connect again while already connecting

Calling `Connect`/`Connect To Server` while a previous connection attempt is still in
progress or already active returns `false` and does nothing — no second, parallel
connection is created. To connect to a different server, call `Disconnect` first.

### Disconnect

```
Disconnect
   Reason : "Player returned to the main menu"
```

`Reason` is arbitrary text for logs and for the `On Disconnected` handler, if you want to
tell a deliberate disconnect apart from a connection drop. Stops the background thread,
closes the socket, clears every subscription. If `b Auto Reconnect` was enabled, the retry
loop stops too.

### IsConnected

Available on all three entry points — the subsystem, the component, and the raw client:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Is Connected ──► (bool) ──► Branch
```

```cpp
if (Nats->IsConnected())
{
    Nats->Publish(TEXT("game.events.ping"), TEXT(""));
}
```

### Get Connection State (NATS Core Client only)

The same state, but with more detail — available only on the raw `NATS Core Client`
(`UNatsClient`), not on the subsystem or the component. The node exists both in Blueprint
(`Is Connected` → a pure pin `Get Connection State` on the client object itself) and in C++:

```cpp
UNatsClient* Client = NewObject<UNatsClient>();
// ...
if (Client->GetConnectionState() == ENatsConnectionState::Connecting)
{
    // The TCP connection or NATS handshake is still in progress
}
```

| Value | What it means |
|---|---|
| `Disconnected` | No active connection. Reasons: `Connect` hasn't been called yet, the connection was dropped, or the first attempt failed |
| `Connecting` | The TCP connection is being established, or the NATS handshake is in progress (usually under a second) |
| `Connected` | Fully ready: you can publish, subscribe, or make requests |

---

## Publishing

### Publish

```
Publish
   Subject  : "game.events.player.join"
   Data     : "{\"player\":\"Alice\"}"
   Reply To : (empty)
```

`Data` is plain text, sent as UTF-8: JSON, plain text, XML — any textual representation.
`Reply To` is only filled in for a hand-rolled request/reply implementation (usually
`Request Async` is used instead — [below](#requestreply)). Returns `true` if the message
was successfully queued for sending; `false` if the client isn't connected.

> `Publish` isn't suitable for binary data (images, compressed data, serialized structs) —
> the text path can corrupt bytes that aren't valid UTF-8. See
> [6. Binary Data](06-Binary-Data.md).

### Publish With Headers

Headers are key-value pairs alongside the data: content type, a trace ID, priority —
anything that's logically separate from the message body itself.

```
Make Map (String → String)
   ["Content-Type"] = "application/json"
   ["Trace-Id"]      = "abc-123-xyz"
   │
   └─► Publish With Headers
          Subject  : "game.events.player.join"
          Data     : "{\"player\":\"Alice\"}"
          Headers  : (the map above)
```

```cpp
TMap<FString, FString> Headers;
Headers.Add(TEXT("Content-Type"), TEXT("application/json"));
Headers.Add(TEXT("Trace-Id"), TEXT("abc-123-xyz"));

Nats->Publish(TEXT("game.events.player.join"), Payload); // without headers
// or, on the same NatsClient available via the Subsystem/Component:
NatsClientInstance->PublishWithHeaders(Subject, Payload, Headers);
```

> **Headers require NATS server 2.2.0 or newer.** If the server doesn't support them,
> `Publish With Headers` returns `false` — check for this if you're working against very
> old server deployments.

The receiver reads headers from the `Headers` field of the `FNatsMessage` struct —
[below](#the-message-struct-fnatsmessage).

---

## Subscribing

### Subscribe / Unsubscribe

```
Subscribe
   Subject : "chat.lobby.*"
```

Supports both wildcard characters — `*` (one token) and `>` (one or more tokens to the
end), covered in detail in
[1. Introduction to NATS](01-Introduction.md#subject-the-messages-address). Each
`Subscribe` call registers a separate subscription; to stop receiving messages on the same
subject, call:

```
Unsubscribe
   Subject : "chat.lobby.*"
```

`Subject` for `Unsubscribe` must **exactly** match the string passed to `Subscribe` —
including any asterisks and the `>` character, if present.

### The message struct (`FNatsMessage`)

Every received message — both in `On Message Received` and as the result of
`Request Async` — arrives as this struct:

| Field | Type | Description |
|---|---|---|
| `Subject` | `FString` | The exact subject the message was published on (not the subscription's pattern) |
| `Data` | `FString` | The message content, decoded as UTF-8 text |
| `Reply To` | `FString` | The reply subject, if the publisher set one (empty for ordinary messages) |
| `Headers` | `TMap<FString, FString>` | Headers, if publishing went through `Publish With Headers` |
| `Payload` | `TArray<uint8>` | The same data, but as exact bytes — see [6. Binary Data](06-Binary-Data.md) |

### The On Message Received event

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On Message Received
            │
            └─► (custom event)
                    Message
                       │
                       ├─► Break Nats Message
                       │       ├─ Subject
                       │       ├─ Data
                       │       └─ Headers
                       │
                       └─► Switch on String (Subject)
```

The event fires for **every** Core message received on any active subscription — if you're
subscribed to several subjects, tell them apart inside the handler by `Message.Subject`
(`Switch on String`, `Contains`, or your own routing logic).

> The event always fires on the game thread, regardless of where the message came from over
> the network — it's safe to touch actors and widgets inside it.

---

## Request/Reply

The pattern is explained conceptually in
[1. Introduction to NATS](01-Introduction.md#requestreply-when-you-need-an-answer); here's
how to use it.

### Request Async

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Request Async
          Subject         : "service.users.get"
          Data            : "{\"id\":123}"
          Timeout Seconds : 5.0
          │
          Callback ──► (bSuccess, Response)
                            │
                            ├─ true  → Print String (Response)
                            └─ false → Print String (Response)   ← holds the reason: "Timeout", "Not connected", etc.
```

The plugin itself creates a temporary inbox (`_INBOX.<guid>`), subscribes to it, publishes
the request with `reply-to` set to that inbox, and waits for the reply — all in a single
call. If no reply arrives within `Timeout Seconds`, `Callback` fires with `bSuccess =
false`, and `Response` contains `"Timeout"`.

C++:

```cpp
Nats->RequestAsync(TEXT("service.users.get"), TEXT("{\"id\":123}"), 5.0f,
    [](bool bSuccess, const FString& Response)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Received: %s"), *Response);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Request failed: %s"), *Response);
        }
    });
```

The callback always runs on the game thread — both `Request Async` (BP) and `RequestAsync`
(the C++ delegate).

### The responder's side

No special node is needed — the responder simply subscribes to the request subject and
publishes its reply to `Message.ReplyTo`:

```
On Message Received (Message → Subject == "service.users.get")
   │
   └─► Publish
          Subject : Message → Reply To
          Data    : "{\"id\":123,\"name\":\"Alice\",\"level\":5}"
```

If the responder publishes nothing back, the requester simply gets a timeout — that's
normal, expected NATS behavior, not a plugin bug.

### RequestAsyncWithHeaders (C++ only)

The same request, but with headers — for example, to pass a request's authorization token
separately from the body:

```cpp
TMap<FString, FString> Headers;
Headers.Add(TEXT("Authorization"), TEXT("Bearer abc123"));

Nats->RequestAsyncWithHeaders(TEXT("service.users.get"), TEXT("{\"id\":123}"), Headers, 5.0f,
    [](bool bSuccess, const FString& Response)
    {
        // ...
    });
```

For a Blueprint version with headers, use `Publish With Headers` on the responder's side
together with ordinary `Request Async` on the requester's side; NATS doesn't distinguish
"a request with headers" from "an ordinary request with headers" — it's the same publish
with `reply-to`.

### Getting the full response object (C++ only)

`RequestAsync` returns only the reply's `Data` as a string. If you also need the headers or
the `ReplyTo` of the reply itself (relevant, for example, for some JetStream pull-consumer
scenarios — [9. JetStream: Consumers](09-JetStream-Consumers.md)), use
`RequestAsyncWithMessage`:

```cpp
Nats->RequestAsyncWithMessage(TEXT("service.users.get"), TEXT("{\"id\":123}"), 5.0f,
    [](bool bSuccess, const FNatsMessage& ResponseMessage)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Data: %s, Headers: %d"),
                *ResponseMessage.Data, ResponseMessage.Headers.Num());
        }
    });
```

---

## Connection events

| Event | When it fires |
|---|---|
| **On Connected** | Right after a successful NATS handshake. This is the correct place for `Subscribe` |
| **On Disconnected** | The connection was dropped: either by your own `Disconnect` call, a network failure, or the server stopping. `Reason` describes why |
| **On Error** | An error at any stage — from a failed connection to a server-side protocol error |

All three always fire on the game thread.

If `b Auto Reconnect` is enabled (the default —
[3. Configuration](03-Configuration.md#reconnection)), after `On Disconnected` (not caused
by your own `Disconnect`) the subsystem itself starts retrying the connection to the same
server; another `On Connected` fires once the connection is restored — subscriptions made
in the `On Connected` handler fire again automatically, since the server doesn't remember
subscriptions from a dropped connection.

---

**Next:** [6. Binary Data](06-Binary-Data.md)
