*🇬🇧 English | [🇺🇦 Українська](../uk/02-Quick-Start.md)*

[← Back to contents](README.md)

# 2. Quick Start

The goal of this chapter is to connect to a server, publish your first message and get it
back within ten minutes, without writing a single line of C++.

---

## Step 0. A server to test against

The plugin is only a client: it needs a separate NATS server to connect to. If you don't
already have one, the fastest way to spin up a local one is a single Docker command:

```bash
docker run -p 4222:4222 -p 8222:8222 nats:latest -js
```

The `-js` flag enables JetStream — you'll need it for chapters 6–9. Port `4222` is the NATS
protocol itself, `8222` is the monitoring page (`http://localhost:8222/varz`), useful for
checking that the server really did come up.

For a more permanent local setup (with JetStream data that survives restarts), the plugin
already ships a ready-made `docker-compose.yml` — details in
[12. Testing and the Local Server](12-Testing-And-Local-Server.md).

---

## Step 1. Installation

1. Copy the `NatsClient/` folder into your project's `Plugins/` directory (or install the
   plugin via Fab/the Epic Games Launcher — then this step isn't needed).
2. Regenerate your project files and build.
3. Make sure the plugin is enabled: **Edit → Plugins → Networking → NATS Message Broker
   Client**.

---

## Step 2. Connection settings

**Project Settings → Plugins → NATS Messaging Client**

For the local server from step 0, the default settings already work:

| Field | Default value | When to change it |
|---|---|---|
| **Server URL** | `127.0.0.1` | Remote server — enter the address or hostname |
| **Port** | `4222` | The server listens on a different port |
| **Credentials → Auth Type** | `None` | The server requires authorization — see [3. Configuration](03-Configuration.md#credentials) |

A full description of every field on this page is in [3. Configuration](03-Configuration.md).

---

## Step 3. First subscribe and publish from Blueprint

```
Event BeginPlay
   │
   ├─► Get Game Instance Subsystem (Nats Client Subsystem) ──┐
   │                                                          │
   │                                                          ├─► Connect To Server
   │                                                          │
   │                                                          ├─► Bind Event to On Message Received
   │                                                          │        │
   │                                                          │        └─► Print String (Message → Data)
   │                                                          │
   │                                                          └─► Bind Event to On Connected
   │                                                                   │
   │                                                                   └─► Subscribe
   │                                                                          Subject : "game.events.>"
```

**Get Game Instance Subsystem** is a standard Unreal node: right-click in the graph →
type "Get Game Instance Subsystem" → pick the **Nats Client Subsystem** class. The
subsystem is the single recommended entry point to the plugin: it lives for the whole
game, holds the connection, and reconnects on its own after a disconnect. You don't need to
create a client manually or store it in a variable.

Subscribe **specifically to `On Connected`**, not right after `Connect To Server`: the call
itself only starts the connection (the TCP handshake and the NATS handshake take a fraction
of a second), and `Subscribe` called before the connection actually exists does nothing.

Now let's publish a message — on the same subject, so we get our own message back:

```
[any event, e.g. a button press]
   │
   └─► Get Game Instance Subsystem (Nats Client Subsystem)
          │
          └─► Publish
                 Subject : "game.events.player.join"
                 Data    : "{\"player\":\"Alice\"}"
```

Press the button — a Print String will appear in the logs with the data you just published:
the subscription on `game.events.>` captured the subject `game.events.player.join` under
the `>` wildcard rule (see [1. Introduction to NATS](01-Introduction.md#subject-the-messages-address)).

---

## Step 4. The same example in C++

```cpp
#include "NatsClientSubsystem.h"

void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();

    Nats->OnConnected.AddDynamic(this, &AMyGameMode::HandleConnected);
    Nats->OnMessageReceived.AddDynamic(this, &AMyGameMode::HandleMessage);

    Nats->ConnectToServer();
}

void AMyGameMode::HandleConnected()
{
    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->Subscribe(TEXT("game.events.>"));
}

void AMyGameMode::HandleMessage(const FNatsMessage& Message)
{
    UE_LOG(LogTemp, Log, TEXT("Received %s: %s"), *Message.Subject, *Message.Data);
}

void AMyGameMode::OnPlayerJoined(const FString& PlayerName)
{
    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->Publish(TEXT("game.events.player.join"),
        FString::Printf(TEXT("{\"player\":\"%s\"}"), *PlayerName));
}
```

`HandleConnected` and `HandleMessage` must be `UFUNCTION()` so `AddDynamic` can bind to
them — this is a standard Unreal requirement for dynamic delegates, not something specific
to the plugin.

---

## Step 5. Verification right from the editor

On the same **Project Settings → Plugins → NATS Messaging Client** page, there are two
buttons at the top:

| Button | What it does |
|---|---|
| **Test Connection** | Connects to the server using the current settings and shows a notification: succeeded or not |
| **Test JetStream** | The same, plus confirms JetStream is enabled on the server |

Handy to check your configuration right after step 2, before you write a single graph.

---

## Step 6. Request/Reply in one call

When you need an actual reply rather than just a subscription — the pattern described in
[1. Introduction to NATS](01-Introduction.md#requestreply-when-you-need-an-answer) — the
plugin hides all the inbox machinery behind a single node:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Request Async
          Subject          : "service.users.get"
          Data             : "{\"id\":123}"
          Timeout Seconds  : 5.0
          │
          Callback → (bSuccess, Response)
                         │
                         ├─ true  → Print String (Response)
                         └─ false → Print String ("Timeout or error")
```

The responder is a separate client (perhaps your dedicated server) that's subscribed to
`service.users.get` and publishes its reply to `Message.ReplyTo`:

```
On Message Received (Subject == "service.users.get")
   │
   └─► Publish
          Subject : Message → Reply To
          Data    : "{\"id\":123,\"name\":\"Alice\"}"
```

A full breakdown, including a binary-data variant, is in
[5. Core Messaging](05-Core-Messaging.md#requestreply).

---

## Step 7. Your first JetStream stream

The publish and subscribe above are Core mode: no subscriber at the moment of publication,
no message. If data can't be lost, you need JetStream —
[1. Introduction to NATS](01-Introduction.md#core-vs-jetstream-two-modes-of-the-same-server).

```
[after On Connected]
   │
   └─► Get JetStream
          │
          └─► Get Streams Manager
                 │
                 └─► Create Stream
                        Config → Name     : "ORDERS"
                        Config → Subjects : ["orders.>"]
                        │
                        └─ bSuccess → Get Publisher → Publish Message
                                         Subject : "orders.created"
                                         Data    : "{\"id\":1}"
```

Now `orders.created` is stored on the server regardless of whether anyone is subscribed at
this exact moment — you can read it even an hour later by creating a consumer. A full
breakdown of streams, consumers (push and pull) and the Key-Value store is in chapters 6–9.

---

## Common early stumbling blocks

| Symptom | Cause |
|---|---|
| `On Connected` never fires | The server is unreachable at the given address/port, or was started without `-js` while you expect JetStream. Check with the **Test Connection** button |
| `Subscribe` brings nothing | The subscribe call happened before `On Connected` — `IsConnected()` was `false` at the time of the call |
| Publish returns `false` | The client isn't connected yet. Check `IsConnected` before publishing |
| `Request Async` always times out | The responder isn't publishing back to `Message.ReplyTo`, or is subscribed to a different subject |
| Data with Cyrillic characters or emoji looks corrupted on the receiving side | Unlikely with this version of the plugin (2.1) — bytes are transmitted exactly; check that the receiver itself isn't truncating/re-encoding the string |

---

**Next:** [3. Configuration](03-Configuration.md)
