*🇬🇧 English | [🇺🇦 Українська](../uk/06-Binary-Data.md)*

[← Back to contents](README.md)

# 6. Binary Data

`Publish` and `Data` from the previous chapter are great for text — JSON, plain strings.
But not everything you need to send is text: images, compressed data, serialized structs,
audio. For this, the plugin has a parallel set of operations that works with exact bytes —
`TArray<uint8>` instead of `FString`.

---

## Why not just Publish

`FString` in Unreal is text, and somewhere along the `Data`/`Publish` path it's treated
exactly as text (UTF-8 encoding on send, decoding on receive). For ordinary strings this is
invisible and correct. But an arbitrary binary blob — a compressed file, an image, a
serialized struct — isn't necessarily valid UTF-8: representing such data directly as text
byte-for-byte can lose or corrupt part of the content.

**The rule is simple:** if the data was authored by a human in a language (JSON, a plain
string, XML), `Publish`/`Data` is right. If the data is the output of serialization,
compression, or a file from disk, use `Publish Bytes`/`Payload`.

---

## Sending: Publish Bytes

```
Publish Bytes
   Subject   : "game.state.snapshot"
   Payload   : (byte array)
   Reply To  : (empty)
```

```cpp
TArray<uint8> Bytes = SerializeGameState();
Nats->PublishBytes(TEXT("game.state.snapshot"), Bytes);
```

Every byte goes onto the wire exactly as it was in the array — including zero bytes inside
the data. The signature is identical to the text `Publish`, just with `Data` replaced by
`Payload`.

### Publish Bytes With Headers

The same binary path, but with headers — for example, to indicate the content format
alongside the bytes themselves:

```
Make Map (String → String)
   ["Content-Type"] = "application/octet-stream"
   │
   └─► Publish Bytes With Headers
          Subject : "game.state.snapshot"
          Payload : (byte array)
          Headers : (the map above)
```

---

## Receiving: the Payload field

Every received message (both in `On Message Received` and in a request's result) already
contains the bytes — nothing needs converting, `Payload` is always filled in regardless of
which method was used to publish the data:

```
On Message Received
   │
   └─► Break Nats Message
           │
           └─ Payload ──► (use as TArray<uint8>)
```

```cpp
void AMyActor::HandleMessage(const FNatsMessage& Message)
{
    if (Message.Subject == TEXT("game.state.snapshot"))
    {
        ApplyGameState(Message.Payload);
    }
}
```

`Data` is filled in too — with the same content, decoded as UTF-8 text. For truly binary
data this field doesn't make sense (a text representation of arbitrary bytes is usually
unreadable or even partially lost) — just ignore it and use `Payload`.

---

## Text ↔ bytes manually

Sometimes you need to explicitly convert a string to bytes (for example, to put text JSON
alongside a binary blob in a single message of your own format) or the other way around.
Two pure nodes in the plugin's library:

```
"Hello" ──► String To UTF-8 Bytes ──► (TArray<uint8>)
```

```
(TArray<uint8>) ──► UTF-8 Bytes To String ──► "Hello"
```

```cpp
TArray<uint8> Bytes = UNatsBlueprintLibrary::StringToUtf8Bytes(TEXT("Hello"));
FString Text = UNatsBlueprintLibrary::Utf8BytesToString(Bytes);
```

Both work correctly with non-Latin characters: each Cyrillic character in UTF-8 takes two
bytes, and the conversion is exact in both directions — what you wrote is what you read
back.

---

## Request/Reply with binary data

The same request/reply pattern as in [5. Core Messaging](05-Core-Messaging.md#requestreply),
but for binary content — the reply is returned as the full `FNatsMessage` struct rather than
just text, since the reply's meaning lives in `Payload`:

```
Request Bytes Async
   Subject         : "service.avatar.generate"
   Payload         : (input parameters as bytes)
   Timeout Seconds : 10.0
   │
   Callback ──► (bSuccess, Response)
                    │
                    ├─ true  → Response → Payload  (the generated image)
                    └─ false → Response → Data     (the reason: "Timeout", etc.)
```

```cpp
Nats->RequestBytesAsync(TEXT("service.avatar.generate"), RequestBytes, 10.0f,
    [](bool bSuccess, const FNatsMessage& Response)
    {
        if (bSuccess)
        {
            SaveAvatarTexture(Response.Payload);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Failed: %s"), *Response.Data);
        }
    });
```

On failure (timeout, no connection), `Response.Data` holds the reason as text, and
`Response.Payload` is empty — consistent with how `Request Async` returns the reason in
`Response` when `bSuccess = false`.

---

## Practical examples

### Send a save game

```cpp
TArray<uint8> SaveData;
UGameplayStatics::SaveGameToMemory(SaveGameObject, SaveData);
Nats->PublishBytes(TEXT("player.save.upload"), SaveData);
```

### Send a texture (e.g. a screenshot)

```
Export To Bytes (Render Target)
   │
   └─► Publish Bytes
          Subject : "screenshots.upload"
          Payload : (from Export To Bytes)
```

### Pack several values into one binary blob (C++)

When you need to send several fields more compactly than JSON — your own binary
serialization via `FMemoryWriter`/`FMemoryReader`, the standard Unreal way:

```cpp
// Sender
TArray<uint8> Bytes;
FMemoryWriter Writer(Bytes);
int32 PlayerId = 42;
FVector Position = GetActorLocation();
Writer << PlayerId;
Writer << Position;

Nats->PublishBytes(TEXT("game.player.position"), Bytes);

// Receiver
void AMyActor::HandleMessage(const FNatsMessage& Message)
{
    if (Message.Subject == TEXT("game.player.position"))
    {
        FMemoryReader Reader(Message.Payload);
        int32 PlayerId;
        FVector Position;
        Reader << PlayerId;
        Reader << Position;
    }
}
```

The field order on write and on read must match exactly — this is low-level byte-for-byte
serialization, with no self-describing format.

---

## JetStream and Key-Value with binary data

The same "a Bytes variant alongside the text one" principle applies to JetStream too:

- **JetStream Publisher** — `Publish Bytes`, `Publish Bytes With Headers`,
  `Publish Bytes With Message ID`, `Publish Bytes With Expected Sequence` — details in
  [8. JetStream: Publishing](08-JetStream-Publisher.md#bytes-variants-of-publishing).
- **Key-Value** — `KV Put Bytes`, `KV Put Bytes With Revision`, `KV Create Bytes`, and a
  value you read back is returned as both text (`Value`) and bytes (`ValueBytes`) at once —
  details in [10. JetStream: Key-Value](10-JetStream-KeyValue.md#binary-values).

---

**Next:** [7. JetStream: Streams](07-JetStream-Streams.md)
