# NATS Client

<!-- last-synced:start -->
<p style="text-align: right; font-size: .75rem; opacity: .7;"><em>Docs last synced: 2026-09-17 15:52 UTC</em></p>
<!-- last-synced:end -->

Native [NATS](https://nats.io) message broker client for **Unreal Engine 5.8** — Core
(publish/subscribe, request/reply) and JetStream (streams, consumers, Key-Value), from
Blueprint and from C++. No third-party libraries: the NATS protocol is implemented directly
over Unreal's own sockets.

| | English | Українська |
|---|---|---|
| **Full** — the complete plugin | [Guide](Full/en/README.md) | [Посібник](Full/uk/README.md) |

## What it covers

Publish/Subscribe with subject wildcards, Request/Reply with an automatic reply inbox,
message headers, and automatic reconnection to the last server after a disconnect — all
available from Blueprint and C++ alike. On top of that, JetStream adds persistent streams,
push and pull consumers with explicit acknowledgment, a Key-Value store with revisions and
real-time Watch, and exact byte-for-byte binary data throughout (publish, request/reply and
Key-Value all have `Bytes` variants).

## Where to start

Three nodes get you from a fresh install to a message sent and received back:

```
Get Game Instance Subsystem (Nats Client Subsystem)  →  Connect To Server  →  Publish
```

Configure the connection once under **Project Settings → Plugins → NATS Messaging Client**,
press **Test Connection**, and the same client is available everywhere in the project.

The chapter most worth reading before shipping anything is
**[Credentials](Full/en/04-Credentials-And-Secrets.md)** (Ukrainian:
[Облікові дані](Full/uk/04-Credentials-And-Secrets.md)) — where credentials belong for
testing versus production, and why anything embedded in a shipped client can be extracted
regardless of how it's stored.
