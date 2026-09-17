*🇬🇧 English | [🇺🇦 Українська](../uk/13-FAQ.md)*

[← Back to contents](README.md)

# 13. FAQ

Short answers to the most frequently asked questions, with a link to the chapter where each
one is covered in depth. If an answer here contradicts a chapter, the chapter is right.

---

## Getting started

### How many nodes does it take to publish my first message?

Three calls:

```
Get Game Instance Subsystem (Nats Client Subsystem) → Connect To Server → (after On Connected) → Publish
```

You don't need to create a client or store it in a variable — the subsystem does that for
you. See [2. Quick Start](02-Quick-Start.md).

### Do I need C++?

No. Every operation — Core, JetStream, Key-Value, binary data — is available as a node. C++
is only needed for what's structurally impossible in Blueprint: for example, getting the
full response object of a request via `RequestAsyncWithMessage` (the ordinary `Request
Async` in BP returns only the reply's text). See
[5. Core Messaging](05-Core-Messaging.md#getting-the-full-response-object-c-only).

### Do I need any third-party libraries or the official NATS SDK?

No. The plugin implements the NATS protocol directly on top of Unreal's standard modules
(`Sockets`, `Networking`) — nothing extra to install.

### Subsystem, component, or NATS Core Client — which should I choose?

The subsystem is the default choice for most projects: one shared message connection for
the whole game, with no manual lifecycle management. The component is for when a connection
logically belongs to a specific actor. The raw client is for advanced scenarios. The
comparison table is in
[5. Core Messaging](05-Core-Messaging.md#three-ways-to-use-the-plugin).

### Can I use the plugin without JetStream?

Yes, completely. Publish/Subscribe/Request-Reply (Core mode) work independently of whether
JetStream is enabled on the server. JetStream is a separate, optional layer for persisting
messages — details in
[1. Introduction to NATS](01-Introduction.md#core-vs-jetstream-two-modes-of-the-same-server).

---

## Configuration and credentials

### Do credentials entered in Project Settings end up in the packaged game?

Yes, in plain form — they go into `DefaultGame.ini` along with the build. There's a separate,
safe override for testing in the editor; for a production game or a dedicated server, fetch
the data at runtime. Both options, with examples, are in
[4. Credentials: How Not to Store Secrets in the Client](04-Credentials-And-Secrets.md).

### How do I test in the editor with real credentials without entering them in Project Settings?

**Editor Preferences → Plugins → NATS Messaging Client (Editor Only)** — a separate page
whose values live only on your machine (`Saved/Config/`) and aren't part of the build. See
[4. Credentials → Editor Only credentials](04-Credentials-And-Secrets.md#editor-only-credentials-for-testing-in-the-editor).

### What do I put in Auth Type if the server has no authorization at all?

`None` — the default value. The other fields of the `Credentials` struct are ignored in
that case.

### Is TLS (`tls://`) supported?

No, in the current version connections are unencrypted only. If you need channel
encryption, use a VPN, an SSH tunnel, or TLS termination at the network infrastructure
level in front of the NATS server.

### Is full decentralized NATS authentication (JWT + NKey) supported?

Not fully. The plugin passes your JWT token in the `jwt` field of the `CONNECT` command, but
doesn't sign the server's challenge (nonce) via NKey — meaning it doesn't implement the full
decentralized NATS authentication protocol. It fits servers that accept a JWT directly (for
example, via an
[auth callout](https://docs.nats.io/running-a-nats-service/configuration/securing_nats/auth_callout)).
More detail — [3. Configuration](03-Configuration.md#credentials).

---

## Messages and subjects

### What's the difference between `*` and `>`?

`*` is exactly one subject token. `>` is one or more tokens, and only at the end of a
pattern. Examples and a table — [1. Introduction to NATS](01-Introduction.md#subject-the-messages-address).

### Can I subscribe to several subjects in one call?

No — each `Subscribe` accepts one string (possibly a pattern). Several unrelated subjects
means several separate `Subscribe` calls.

### Are queue groups supported for load balancing?

No, in the current version `Subscribe` doesn't accept a group name — each subscription is
independent and receives every message. If you need to distribute processing across several
workers so that each message reaches only one of them, use a JetStream pull consumer: the
server itself guarantees that the same message won't be handed to two concurrent
`Pull Messages` requests. Example —
[9. JetStream: Consumers → Task Queue Example](09-JetStream-Consumers.md#example-a-task-queue-work-queue).

### What's the maximum message size?

Limited by the server, via its `max_payload` config setting (1 MB on the local server
bundled with the plugin). Starting with version 2.1, the client itself raises its
**receive** limit if the server advertises a larger `max_payload`; the client doesn't
impose its own limit on **sending** — an oversized message will be rejected by the server
with a protocol error, visible in `On Error`.

---

## JetStream

### How does a stream differ from an ordinary subscription?

An ordinary (Core) subscription only receives what's published while it's active — nothing
is stored. A stream stores messages on the server; you can read them even an hour later by
creating a consumer. Details —
[1. Introduction to NATS](01-Introduction.md#core-vs-jetstream-two-modes-of-the-same-server).

### Push or pull consumer — which should I choose?

Push — when you need an instant, real-time reaction. Pull — when the pace of processing
should be controlled by the consumer itself (batch processing, task queues). The comparison
table — [9. JetStream: Consumers](09-JetStream-Consumers.md#push-vs-pull-which-to-choose).

### Do I have to create streams and buckets through Project Settings?

No, that's purely a convenience. You can also create them with a direct `Create Stream`/
`Create KV Bucket` call in code or a graph, whenever you like once JetStream has become
available. Project Settings is a declarative way for anyone who'd rather have it all in one
place. See [3. Configuration](03-Configuration.md#jetstream-auto-creating-resources).

### What happens if I don't acknowledge a message in time (AckWait)?

The server considers the message unprocessed and delivers it again — `NumDelivered`
increases. If processing is systematically slower than the current `AckWait`, increase that
value in the consumer's configuration rather than acknowledging prematurely. Details —
[9. JetStream: Consumers](09-JetStream-Consumers.md#acknowledgment-ack--nak--term).

---

## Errors and networking

### On Connected never fires

The server is unreachable at the given address or port. Check with the **Test Connection**
button in Project Settings and look at the `On Error` event — it contains the reason. See
[11. Errors and Diagnostics](11-Errors-And-Diagnostics.md#common-core-messages).

### Request Async always times out

The most common cause: the responder isn't publishing back to `Message.ReplyTo`, or is
subscribed to a different subject than the one the request is sent to. See
[5. Core Messaging → Request/Reply](05-Core-Messaging.md#requestreply).

### Does the plugin reconnect on its own after a connection drop?

Yes, if `b Auto Reconnect` is enabled (the default) — the subsystem and the component retry
the connection themselves, **to the same server** they last connected to. The raw
`NATS Core Client` doesn't reconnect on its own — see the note in
[5. Core Messaging](05-Core-Messaging.md#three-ways-to-use-the-plugin).

---

## Multiplayer and servers

### Does the plugin replicate anything over Unreal's network?

No. It opens a separate TCP connection to the NATS server from whichever process calls it —
this is completely separate from Unreal's actor replication. The question "how does this
work in multiplayer" reduces to "which process (client, server, both) am I calling the
plugin's nodes from" — that's for you to decide, the same as any other branch of logic
behind `Has Authority`/`Switch Has Authority`.

### In an eight-client session, a message gets published eight times

Likely the `Publish` call is in code that runs on every client locally (for example, in an
actor's `Tick` with no authority check). Guard the call with `Switch Has Authority` if only
the server should be publishing.

### Does the plugin work on a headless dedicated server (`-nullrhi`)?

Yes — the plugin operates at the socket level and doesn't depend on rendering. It works for
`-server` builds and for running in a container without a GPU.

---

## Miscellaneous

### Can I have several independent connections at once?

Yes — each `NATS Client Component` (on different actors) or its own `NATS Core Client`
instance holds its own, fully independent connection: its own address, credentials,
subscriptions.

### Is there a limit on the number of concurrent subscriptions?

The plugin doesn't impose its own limit — only the server does (typically quite high,
thousands of subscriptions per connection). In practice, you're limited more by how many
meaningful subjects your game actually has than by any technical ceiling.

### Can I publish a message before the client is connected (e.g. into a queue)?

No — `Publish`/`Publish Bytes` immediately return `false` if `Is Connected` is `false` at
the moment of the call; the plugin doesn't buffer messages for deferred sending. If you need
exactly that behavior, implement your own queue on the game side and drain it in the
`On Connected` handler.

### Does the plugin's test module (`NatsClientTests`) end up in the packaged game?

No. It's an `Editor`-type module — it's only built for the editor and never ships in a
Development or Shipping game build.

---

**Next:** [Release Notes](Release-Notes.md)
