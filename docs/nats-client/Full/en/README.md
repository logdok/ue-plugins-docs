# NATS Message Broker Client

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

Native NATS client for Unreal Engine — Core (publish/subscribe, request/reply) and
JetStream (streams, consumers, Key-Value), from Blueprint and from C++.

> **Version 2.1** · Runtime · UE 5.8 · [Release Notes](Release-Notes.md)
> Win64 · Mac · Linux · iOS · Android · tvOS
> No third-party libraries: the NATS protocol is implemented directly over Unreal's sockets

---

## Quick start: three nodes

The shortest path from "just installed" to "message sent and received back."

**1. Start a test server** (if you don't already have one):

```bash
docker run -p 4222:4222 -p 8222:8222 nats:latest -js
```

**2. Connect and subscribe**

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   ├─► Connect To Server
   │
   └─► Bind Event to On Connected
            │
            └─► Subscribe   Subject: "game.events.>"
```

You don't need to create a client or store it in a variable — the subsystem does that for
you.

**3. Publish**

```
Get Game Instance Subsystem (Nats Client Subsystem)  →  Publish   Subject: "game.events.player.join"
```

A full step-by-step walkthrough, including verification with buttons right in the editor,
is in [2. Quick Start](02-Quick-Start.md).

---

## Chapters

| Chapter | What it covers |
|---|---|
| [Release Notes](Release-Notes.md) | What's new and fixed in each release |
| [1. Introduction to NATS](01-Introduction.md) | Subject, wildcards, publish/subscribe, request/reply, Core vs. JetStream — the terms everything else relies on |
| [2. Quick Start](02-Quick-Start.md) | Your first connection, first publish and first stream, step by step |
| [3. Configuration](03-Configuration.md) | The Project Settings page, credentials, the actor component, auto-subscribe and JetStream auto-creation |
| [4. Credentials: How Not to Store Secrets in the Client](04-Credentials-And-Secrets.md) | The Editor Only override for testing and getting credentials at runtime for production |
| [5. Core Messaging](05-Core-Messaging.md) | Subsystem, component, or raw client; connecting, Publish/Subscribe, Request/Reply, events |
| [6. Binary Data](06-Binary-Data.md) | `TArray<uint8>` instead of text: images, compressed data, serialized structs |
| [7. JetStream: Streams](07-JetStream-Streams.md) | Creating and managing streams, retention, configuration builders |
| [8. JetStream: Publishing](08-JetStream-Publisher.md) | Publishing with server acknowledgment, deduplication, optimistic concurrency |
| [9. JetStream: Consumers](09-JetStream-Consumers.md) | Push vs. pull, Ack/Nak/Term acknowledgment, a work-queue example |
| [10. JetStream: Key-Value](10-JetStream-KeyValue.md) | Key-value store: revisions, real-time Watch, history |
| [11. Errors and Diagnostics](11-Errors-And-Diagnostics.md) | Error codes, common causes, logs, test buttons |
| [12. Testing and the Local Server](12-Testing-And-Local-Server.md) | A ready-made Docker server bundled with the plugin, verifying your own integration, automated tests |
| [13. FAQ](13-FAQ.md) | Short answers to the most frequently asked questions |

---

## What the plugin can do

**Core.** Publish/Subscribe with subject wildcards (`*`/`>`), Request/Reply with an
automatic reply inbox and timeout, message headers, automatic reconnection to the last
server after a disconnect.

**Authentication.** No auth, Basic (username/password), Token (shared secret), JWT — each
method is available both from the settings page and from code at connect time.

**Binary data.** Publish, request-reply and Key-Value all work with both text and exact
bytes — not a single byte (zero bytes included) is lost or corrupted along the way.

**JetStream: streams.** Persistent message storage with flexible limits (count, size, age),
three retention policies (Limits, Interest, WorkQueue) and configuration builders for C++.

**JetStream: consumers.** Push (the server sends to you) and pull (you request in batches)
consumers, explicit acknowledgment (Ack/Nak/Term), flexible choice of where to start
reading — from the beginning of history, from a specific point, or only new messages.

**JetStream: publishing.** Server acknowledgment of writes with a sequence number,
deduplication by your own message ID, optimistic concurrency via an expected sequence.

**JetStream: Key-Value.** A key-value store with revisions, optimistic locking, an atomic
"create only if it doesn't exist," real-time change watching (including key wildcards) and
version history.

**Three entry points.** A subsystem (one shared connection for the whole game), an actor
component (a connection tied to a specific actor), or a raw connection object for full
control — the same set of operations across all three.

**Diagnostics.** Structured error codes in JetStream, separate log categories for each
layer, connection and JetStream test buttons right in Project Settings.

---

## What's missing

- **TLS connections.** The channel is currently unencrypted — use your network
  infrastructure (VPN, tunnel, TLS termination in front of the server) to protect traffic.
- **Full decentralized JWT + NKey authentication.** The JWT token is passed to the server as
  is, without signing the server's nonce challenge with a private key — details in
  [3. Configuration](03-Configuration.md#credentials).
- **A server list for failover connections (seed list).** Each connection targets one
  specific address; on disconnect the client retries that same address rather than falling
  over to another server in the cluster.
- **Queue groups at the Core-subscription level.** To distribute load across multiple
  handlers, use a JetStream pull consumer — details in the
  [FAQ](13-FAQ.md#are-queue-groups-supported-for-load-balancing).
