*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

[← Back to contents](README.md)

# Release Notes

A short summary of every release: what's new, what's fixed, and what to watch for when
upgrading. Newest release first.

---

## 2.1

A release about reliability and binary data. Connecting and reconnecting are more stable,
messages of any content — from Cyrillic text to arbitrary bytes — travel without
corruption, and large messages arrive hundreds of times faster.

### New

- **Binary data.** Send and receive byte arrays (`TArray<uint8>`) from Blueprint and C++:
  - `Publish Bytes` and `Request Bytes Async` — on the client, the subsystem and the
    component;
  - `Publish Bytes` in JetStream, including with a Message ID, headers and an expected
    sequence;
  - `KV Put Bytes`, `KV Put Bytes With Revision` and `KV Create Bytes` in the key-value
    store.

  Received bytes are available in the message's `Payload` field and the KV entry's
  `ValueBytes` field.
- `String To UTF-8 Bytes` and `UTF-8 Bytes To String` nodes for converting text to bytes and
  back.
- The server address can now be given as a hostname (`nats.example.com`), not just an IP
  address.
- `Get Connection State` reports whether the client is disconnected, connecting, or already
  connected.
- A separate **Editor Preferences → Plugins → NATS Messaging Client (Editor Only)** page —
  credentials for testing in the editor that never make it into `DefaultGame.ini` or the
  packaged game.

### Fixed

- Threading and memory bugs that could throw exceptions while starting and stopping the
  game in the editor, particularly under a debugger.
- Cyrillic text and other non-ASCII characters broke the exchange with the server.
- Zero bytes in the data truncated messages.
- After a server restart, the client failed to notice the disconnect, so automatic
  reconnection didn't kick in.
- Reconnection targeted the address from Project Settings even if the initial connection
  had used a different one.
- The subsystem ignored the `bAutoReconnect` and `ReconnectIntervalSeconds` settings.
- A repeated `Connect` call while already connecting could corrupt memory.
- `Disconnect` called from a message handler could permanently freeze the game.
- Passwords and tokens containing special characters (`"`, `\`) failed authorization.
- `KV Put With Revision` didn't actually check the revision.
- `KV Create` didn't guarantee that only one client would create the key.
- `KV Watch` for every key in a bucket didn't receive updates.
- `Pull Messages` could return empty server housekeeping messages instead of real ones.
- Every JetStream publish left a stray subscription on the server.
- If the plugin was installed from Fab, an error was logged on every connection.
- The "Test Connection" and "Test JetStream" buttons in the plugin's settings now work more
  reliably.

### Performance

- Receiving data is faster: a 900 KB message now arrives in 0.03s instead of 23s.
- Messages are delivered every frame, rather than once every 100ms.

### Known limitations

- After `KV Delete` in a bucket with history (`History` greater than 1), `KV Get` returns
  the key's previous value instead of "key not found."
- TLS connections aren't supported.

### Upgrading

- Projects with C++ code need to be rebuilt.
- The `On Disconnected` and `On Error` events now always arrive on the game thread.
- If `bAutoReconnect` is disabled in Project Settings, the subsystem no longer reconnects on
  its own.
- `KV Put With Revision` now returns an error if the key's revision has already changed —
  as documented. If your code relied on the value always being written, use `KV Put`
  instead.

---

[← Back to contents](README.md)
