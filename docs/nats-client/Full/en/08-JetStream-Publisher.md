*🇬🇧 English | [🇺🇦 Українська](../uk/08-JetStream-Publisher.md)*

[← Back to contents](README.md)

# 8. JetStream: Publishing

Publishing into JetStream differs from an ordinary `Publish`
([5. Core Messaging](05-Core-Messaging.md#publishing)) mainly in that **the server
acknowledges the write**: the call doesn't just queue the message for sending, it waits for
the server to confirm the message was actually stored in the stream, and returns a sequence
number.

---

## Publish Message

```
Get JetStream
   │
   └─► Get Publisher
          │
          └─► Publish Message
                 Subject         : "orders.created"
                 Data            : "{\"id\":1,\"total\":99.99}"
                 Timeout Seconds : 5.0
                 │
                 └─ bSuccess → PubAck
                                  ├─ Stream     : "ORDERS"
                                  ├─ Sequence   : 1
                                  └─ Duplicate  : false
```

```cpp
UNatsJetStreamPublisherImpl* Publisher = Nats->GetJetStream()->Publisher();

Publisher->Publish(TEXT("orders.created"), TEXT("{\"id\":1,\"total\":99.99}"),
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (Result.IsSuccess())
        {
            UE_LOG(LogTemp, Log, TEXT("Stream=%s Seq=%d"), *Result.Value.Stream, Result.Value.Sequence);
        }
        else
        {
            UE_LOG(LogTemp, Error, TEXT("Publish failed: %s"), *Result.Error.Message);
        }
    });
```

If no stream captures the `Subject`, the operation fails (the server doesn't respond, and
the result comes back with `EJetStreamErrorCode::ServerError`), unlike an ordinary
`Publish`, which in that case simply does nothing. This is expected: JetStream guarantees
**persistence**, so publishing into nothing is a configuration mistake, not a normal
fire-and-forget scenario.

### FJetStreamPubAck

| Field | Description |
|---|---|
| `Stream` | Which stream stored the message |
| `Sequence` | The sequence number assigned to the message in the stream |
| `Duplicate` | `true` if the message was recognized as a duplicate (see [Message ID](#deduplication-publish-with-message-id) below) |
| `Domain` | The JetStream domain in multi-tenant deployments (usually empty) |

---

## Deduplication: Publish With Message ID

The network is unreliable: a client can fail to receive an acknowledgment even when the
server has already stored the message, and retry the publish "just in case." To keep that
retry from creating a duplicate order, provide your own unique message ID:

```
Publish With Message ID
   Subject     : "orders.created"
   Data        : "{\"id\":1,\"total\":99.99}"
   Message Id  : "order-1-attempt"
```

```cpp
Publisher->PublishWithMsgId(TEXT("orders.created"), Data, TEXT("order-1-attempt"),
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (Result.IsSuccess() && Result.Value.Duplicate)
        {
            // The server has already seen this Message Id — no new message was added,
            // Sequence points at the existing record.
        }
    });
```

If a message with the same `Message Id` was already received within the server's
deduplication window (typically 2 minutes), the server returns `Duplicate = true` and does
**not** create a second record — the `Sequence` in the response points at the existing
message. This makes a retry safe: code can simply repeat the publish whenever it's unsure
about the outcome, without risking duplicated data.

---

## Optimistic concurrency: Publish With Expected Sequence

When several publishers might write to the same subject concurrently and order matters,
`Publish With Expected Sequence` gives you a "write only if I know the stream's last state"
publish:

```
Publish With Expected Sequence
   Subject           : "orders.created"
   Data              : (new order)
   Expected Last Seq : 41      ← I believe the last message in the stream is #41
```

```cpp
Publisher->PublishWithExpectedSeq(Subject, Data, 41,
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (!Result.IsSuccess())
        {
            // Someone else published a message between reading the state and this call —
            // ExpectedLastSeq no longer matches. Read the current LastSeq
            // (Get Stream Info) and decide whether to retry.
        }
    });
```

A typical loop: read `Get Stream Info` → `State.LastSeq`, attempt the publish with that
value; on failure, read the current `LastSeq` again and decide whether to retry.

---

## Bytes variants of publishing

Each of the four methods above has a binary counterpart — the same principle as in
[6. Binary Data](06-Binary-Data.md): bytes are transmitted exactly, without text encoding.

| Text | Bytes |
|---|---|
| `Publish Message` | `Publish Bytes` |
| `Publish With Headers` | `Publish Bytes With Headers` |
| `Publish With Message ID` | `Publish Bytes With Message ID` |
| `Publish With Expected Sequence` | `Publish Bytes With Expected Sequence` |

```
Publish Bytes
   Subject : "assets.uploaded"
   Payload : (byte array, e.g. a compressed file)
```

```cpp
Publisher->PublishBytes(TEXT("assets.uploaded"), CompressedBytes, Callback);
```

---

## Publishing with headers

```
Make Map (String → String)
   ["Content-Type"] = "application/json"
   │
   └─► Publish With Headers
          Subject : "orders.created"
          Data    : "{\"id\":1}"
          Headers : (the map above)
```

Headers in JetStream work the same way as in Core
([5. Core Messaging](05-Core-Messaging.md#publish-with-headers)) — these are ordinary NATS
message headers, JetStream just stores them alongside the data.

---

## The On Publish Ack event

Besides the result returned by each individual call's callback, there's a general event —
handy for centralized monitoring of every publish rather than handling each one separately:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On JetStream Pub Ack
            │
            └─► (bSuccess, PubAck) → update a UI counter, log it
```

Fires for **every** successful and unsuccessful publish through the Publisher, regardless of
exactly where it was called from.

---

## Practical example: reliably sending an order with retry

```cpp
void AOrderService::SubmitOrder(const FString& OrderId, const FString& OrderJson)
{
    UNatsJetStreamPublisherImpl* Publisher = Nats->GetJetStream()->Publisher();

    // OrderId as the Message Id: a retry (e.g. after a network timeout)
    // is safe — there won't be a duplicate.
    Publisher->PublishWithMsgId(TEXT("orders.created"), OrderJson, OrderId,
        [this, OrderId](TJetStreamResult<FJetStreamPubAck> Result)
        {
            if (Result.IsSuccess())
            {
                UE_LOG(LogTemp, Log, TEXT("Order %s: seq=%d, duplicate=%s"),
                    *OrderId, Result.Value.Sequence, Result.Value.Duplicate ? TEXT("yes") : TEXT("no"));
            }
            else if (Result.Error.Code == EJetStreamErrorCode::Timeout)
            {
                // A network issue — the same Message Id is safe to retry.
                RetrySubmitOrder(OrderId);
            }
        });
}
```

---

**Next:** [9. JetStream: Consumers](09-JetStream-Consumers.md)
