*🇬🇧 English | [🇺🇦 Українська](../uk/01-Introduction.md)*

[← Back to contents](README.md)

# 1. Introduction to NATS

This chapter isn't about the plugin. It's about NATS itself: what kind of system it is, what
vocabulary it speaks, and why it has two different modes — Core and JetStream. If you're
already familiar with NATS, feel free to skip straight to [2. Quick Start](02-Quick-Start.md).
If not, read this chapter in full: every chapter after it relies on the terms introduced here.

---

## What NATS is

NATS is a message broker: a separate program (a server) that accepts messages from some
clients and hands them off to others. Your game is a client too, connecting to that server
over an ordinary TCP connection.

Why do you need a separate server at all, when actors in Unreal can already call functions
on each other directly? Because NATS solves three problems that direct calls don't:

- **Participants don't need to know about each other.** The game server publishes a
  `game.match.finished` event — and it doesn't care whether there's currently a single
  subscriber. Analytics, a logger and an achievements system will each subscribe to this
  event independently, each in its own game instance or as a separate service.
- **Participants can be in different processes, on different machines.** A player on one
  computer, a dedicated server on another, an analytics service as a third process. The
  NATS server is the single point every one of them connects to.

  > The plugin itself replicates nothing over Unreal's own networking — see the warning in
  > [FAQ → "Can this replace Unreal replication?"](13-FAQ.md#does-the-plugin-replicate-anything-over-unreals-network).
- **The publisher doesn't wait for subscribers.** `Publish` returns control immediately —
  the server distributes a copy of the message to every subscriber on its own, in parallel,
  with no request/reply round trip per subscriber.

## Subject: the message's address

Everything in NATS is addressed by a **subject** — a string of tokens separated by dots:
`game.events.player.join`, `chat.lobby.42`, `orders.created`. It's not a filesystem and not
a URL — just a hierarchical address with no predefined structure: how many tokens there are
and what they mean is entirely up to you, within your own project.

A publisher always specifies an exact subject. A subscriber can specify an exact subject
**or a wildcard pattern** using two special characters:

| Symbol | Meaning | Example | Matches | Doesn't match |
|---|---|---|---|---|
| `*` | Exactly one token | `game.events.*` | `game.events.join` | `game.events.player.join` |
| `>` | One or more tokens, to the end | `game.events.>` | `game.events.join`, `game.events.player.join` | `game.other` |

`>` only makes sense as the last token of a pattern. An example spanning a whole game:

```
game.events.player.join       ← a specific event
game.events.player.*          ← every single-token player event (join, leave, death — not damage.taken)
game.events.>                 ← absolutely everything under game.events, at any depth
```

The same subscriber can receive messages from several publishers on different subjects at
once — just subscribe to a shared pattern.

## Publish/Subscribe: the basic pattern

This is the foundation of all of NATS. A publisher sends a message on a subject; everyone
currently subscribed to that (or a matching) subject gets their own copy.

```
Publisher                    NATS server                  Subscriber A (subject: chat.>)
   │                              │                              │
   ├─ Publish "chat.lobby.1" ────►│                              │
   │                              ├──────── copy ─────────────────►│  received
   │                              │
   │                              │                        Subscriber B (subject: chat.lobby.2)
   │                              │                              │
   │                              │            (did not receive — different subject)
```

The most important consequence of this model: **if there's no subscriber on a subject at
the moment of publication, the message simply disappears.** NATS Core stores nothing and
owes no one anything — it's "fire-and-forget" in the literal sense. A subscriber that joins
a second later gets nothing. If you need a message to wait for a subscriber that hasn't
connected yet, that's exactly why JetStream exists (below).

## Request/Reply: when you need an answer

Publish/Subscribe is a one-way flow. When you specifically need a reply to a particular
request ("give me this player's data by ID"), NATS doesn't add a new primitive — the same
Publish/Subscribe is used more cleverly:

1. The requester creates a temporary subject "inbox," unique to this request.
2. The requester subscribes to this inbox.
3. The requester publishes the request to the target subject, specifying the inbox as
   `reply-to`.
4. The responder, receiving the message, sees `reply-to` and publishes its reply exactly
   there.
5. The requester receives the reply on its own inbox — or a timeout fires if no one replied.

```
Requester                         NATS                          Responder
   │                                │                                │
   ├─ Subscribe "_INBOX.abc123" ───►│                                │
   ├─ Publish "svc.users.get"       │                                │
   │  reply-to="_INBOX.abc123" ────►│──────── delivered ──────────────►│
   │                                │                                │  processed the request
   │                                │◄─── Publish "_INBOX.abc123" ────┤
   │◄──────── delivered ─────────────┤                                │
   │  reply received                │                                │
```

The plugin hides all of this machinery behind a single call — `Request Async` — details in
[5. Core Messaging](05-Core-Messaging.md#requestreply).

## Headers: metadata alongside the data

Besides the data itself, a message can carry **headers** — key-value pairs similar to HTTP
headers: content type, a trace ID, a schema version. Data and headers travel separately, so
you never need to parse headers out of the message body.

## Core vs. JetStream: two modes of the same server

Everything above is **NATS Core**: the simplest, fastest mode, with no persistence beyond
instant delivery to disk or memory. The same server, if you enable it, also offers a second
mode — **JetStream**: a persistence layer on top of Core.

| | **Core** | **JetStream** |
|---|---|---|
| What's stored | Nothing. No subscriber, message lost | Messages land in a **stream** and stay there |
| Who receives it | Only those subscribed **at the moment of publication** | Anyone who creates a **consumer** — even an hour after publication |
| Delivery acknowledgment | None | The consumer acknowledges each message (ACK); no ACK, redelivery |
| Speed | Highest | Slightly slower — the server writes the message before acknowledging the publish |
| Typical use | Live events: player movement, real-time chat, lobby state | Anything that must not be lost: orders, achievements, an event log, a task queue |

It's important to understand: JetStream **doesn't replace** Core, it **adds to** it. The
subject `orders.created` can be published even without any stream at all — nobody just
stores it. As soon as you create a stream configured for this subject, the server starts
storing it — the publishing client code doesn't change at all.

## Stream: where messages are stored

A stream is a named message store, similar to a table in a database. You create a stream
once, specifying:

- **a name** — for example `ORDERS`;
- **the subject(s) it captures** — for example `orders.>`. Anything published on a matching
  subject is automatically captured into the stream for as long as it exists;
- **how long to keep it** — by message count, by size, or by age;
- **where to keep it** — in memory (fast, gone on server restart) or on disk (survives a
  restart).

The publisher knows nothing about streams — it just publishes to a subject. The server
decides on its own whether a stream captures that subject, and if so, stores a copy.

## Consumer: a read cursor into a stream

A stream stores messages, but doesn't send them to anyone on its own — reading requires a
**consumer**, a cursor with its own position in the stream. Several consumers can
independently read the same stream, each from its own position.

There are two delivery modes:

- **Push consumer** — the server sends you messages itself, as soon as they appear. Similar
  to an ordinary Core subscription, just with acknowledgment and a guarantee that nothing
  gets lost.
- **Pull consumer** — you ask yourself: "give me up to 10 messages." Useful when the
  consumer itself should control the pace of processing — for example, a task queue that
  the server processes in batches.

After processing each message, the consumer must **acknowledge** it (ACK). Without
acknowledgment within the configured time, the server considers the message unprocessed and
delivers it again.

## Key-Value: a store built on top of a stream

Key-Value (KV) is a layer on top of JetStream that looks like an ordinary "key → value"
dictionary: `Put("player_42", "...")`, then `Get("player_42")`. Under the hood it's the same
stream, where each entry is a separate message on an internal subject shaped like
`$KV.<bucket>.<key>`, and the "latest value of a key" is simply the last message on that
subject. That's where several useful side effects come from:

- **Revisions.** Every write to a key gets an increasing version number — you can check
  whether the key was changed concurrently (optimistic concurrency).
- **Watch.** You can subscribe to changes on a key or a group of keys in real time — this is
  the same Core subscription on an internal subject, the plugin just hides the details.
- **History.** If a bucket is configured to keep several versions, you can read a key's
  previous values.

## What's next

Now that the terminology is in place, [2. Quick Start](02-Quick-Start.md) walks you through
your first connection, first publish and first stream — with the plugin's actual nodes and
code.

---

**Next:** [2. Quick Start](02-Quick-Start.md)
