*🇬🇧 English | [🇺🇦 Українська](../uk/03-Configuration.md)*

[← Back to contents](README.md)

# 3. Configuration

---

## The project settings page

**Project Settings → Plugins → NATS Messaging Client**

This is the main place to configure the plugin — the subsystem (the recommended entry point,
see [5. Core Messaging](05-Core-Messaging.md#three-ways-to-use-the-plugin)) reads these
values every time **Connect To Server** is called. Values are stored in `DefaultGame.ini` —
meaning they go into version control and into the packaged game.

### Connection

| Field | Default | Description |
|---|---|---|
| **Server URL** | `127.0.0.1` | An IP address or hostname, e.g. `nats.example.com`. No scheme (`nats://`) and no port |
| **Port** | `4222` | The server's TCP port. `4222` is NATS's standard port |
| **Client Name** | `UnrealEngine` | The name this client shows up as in server monitoring (`nats-top`, `/varz`) and in its logs. Handy for telling multiple clients apart: e.g. `"GameServer-1"`, `"LobbyService"` |
| **Credentials** | `AuthType = None` | The authentication method — see [below](#credentials) |

> **About `Credentials` on this page: they end up in the packaged game, in plain text.**
> Everything filled into `Credentials` here is stored in `DefaultGame.ini` and ships with the
> build as an ordinary text file. For local development or a server with no sensitive data,
> that's not a problem. For production credentials (a password, a long-lived token) there
> are two safer paths: an **editor override**, so the secret never sits in this page's
> `Credentials` at all — [below](#editor-only-credentials-for-testing) — or fetching them at
> runtime and passing them to `Connect With Credentials`, as shown in
> ["Where to set credentials"](#where-to-set-credentials).

> **About `Server URL`.** You can give it either an IP address or a hostname — the plugin
> resolves the name itself. TLS connections (`tls://`) aren't currently supported — more in
> ["What's missing"](README.md#whats-missing).

### Reconnection

| Field | Default | Description |
|---|---|---|
| **b Auto Reconnect** | `true` | Whether to try reconnecting after a disconnect |
| **Reconnect Interval Seconds** | `5.0` | Pause between attempts, from 1 to 60 seconds |

When `On Disconnected` fires for a reason other than your own `Disconnect` call, the
subsystem itself starts retrying the connection to **the same server** it was last connected
to (not necessarily the one configured here, if you connected via `Connect` with a different
address). If `b Auto Reconnect` is disabled, you'll need to call reconnect yourself after a
disconnect.

### Auto-subscribe

| Field | Description |
|---|---|
| **Auto Subscribe Subjects** | A list of subjects (supporting `*`/`>` wildcards) the subsystem subscribes to right after connecting — before `On Connected` |

Example: `["chat.>", "system.notifications"]` — subscribes to all chat and system
notifications without a single `Subscribe` call in your graphs.

### JetStream: auto-creating resources

| Field | Description |
|---|---|
| **Auto Create Streams** | An array of `Jet Stream Stream Config` — streams the subsystem will create (or confirm exist) right after JetStream becomes available |
| **Auto Create KV Buckets** | The same for Key-Value buckets |

This is a declarative way to guarantee that the infrastructure you need exists before your
game logic starts using it — no need to scatter `Create Stream` calls through your code. If
a stream/bucket with that configuration already exists, it's simply reused — no error. Full
field descriptions are in
[7. JetStream: Streams](07-JetStream-Streams.md#stream-configuration) and
[10. JetStream: Key-Value](10-JetStream-KeyValue.md#bucket-configuration).

> If the server has no JetStream, these two fields are simply ignored — the error goes to
> the log, but the Core connection still works as usual.

---

## Credentials

`FNats Credentials` is a single structure for all four authentication methods, used in the
subsystem, the component, and directly on `UNatsClient`. In the editor, the structure's
fields hide themselves depending on the chosen `Auth Type` — you only need to fill in
what's actually used.

| Auth Type | Which fields to fill in | When it applies |
|---|---|---|
| **None** | — | Local development, a server with no authorization, or access restricted at the network level (a firewall) |
| **Basic** | `Username`, `Password` | The server is set up for plain user accounts |
| **Token** | `Token` | One shared secret for all clients — simpler than Basic, since no username is needed |
| **JWT** | `JWTToken` | The token is issued by your own authentication system (a backend, an auth service) |

> **A JWT caveat that matters.** The plugin passes the `JWTToken` value as-is in the `jwt`
> field of the `CONNECT` command — and that's it. It does **not** implement the full
> decentralized NATS authentication protocol via NKey (signing the server's nonce challenge
> with a private key). This fits servers configured to accept a JWT directly — for example,
> via an
> [auth callout](https://docs.nats.io/running-a-nats-service/configuration/securing_nats/auth_callout)
> or your own authorization proxy layer — but not a "classic" NATS account that requires an
> NKey signature. If your server is that kind, authentication will need extra work on the
> client side.

### Where to set credentials

There are three equally valid places — pick one depending on where the data naturally comes
from.

**1. Project Settings** (above) — if the credentials are known at development time and are
the same for every run.

**2. The actor component** — the `Nats Credentials` field on `NATS Client Component`, if the
connection is tied to a specific actor with its own credentials.

**3. Code, at connect time** — when the data comes from outside (for example, after the
player logs in on your backend):

```
[got a token from the backend after login]
   │
   └─► Make Nats Credentials
          Auth Type : Token
          Token     : (from the backend's response)
          │
          └─► Get Game Instance Subsystem (Nats Client Subsystem)
                 │
                 └─► Connect With Credentials
                        Server URL     : "nats.mygame.com"
                        Port           : 4222
                        In Credentials : (from above)
```

C++:

```cpp
FNatsCredentials Credentials;
Credentials.AuthType = ENatsAuthType::Token;
Credentials.Token = TokenFromBackend;

UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
Nats->ConnectWithCredentials(TEXT("nats.mygame.com"), 4222, Credentials);
```

There's also **Set Credentials** — remembers the data for the **next** connection, without
touching the current one. Calling it without a following `Connect`/`Connect To Server`
doesn't do anything:

```cpp
Nats->SetCredentials(NewCredentials);
Nats->Disconnect(TEXT("Changing credentials"));
Nats->ConnectToServer(); // now uses the new credentials
```

### Editor Only credentials (for testing)

**Project Settings → Plugins → NATS Messaging Client (Editor Only)** — a separate settings
page next to the main one, for testing in the editor with real credentials without the risk
of accidentally committing them into `DefaultGame.ini`.

| Field | Description |
|---|---|
| **b Use Editor Credentials** | The switch. Off — everything works as usual, using the main page |
| **Credentials** | The same structure type as on the main page — all four authentication methods |

A full breakdown — why this is safe, what exactly it affects (and what it doesn't), and what
to do for a **production** game or dedicated server where the secret can't live even here —
is a dedicated chapter:
[4. Credentials: How Not to Store Secrets in the Client](04-Credentials-And-Secrets.md).

---

## The component: `NATS Client Component`

An alternative to the subsystem — when the connection is logically tied to a specific actor
rather than the whole game (more on choosing between them in
[5. Core Messaging](05-Core-Messaging.md#three-ways-to-use-the-plugin)). Add the component
to an actor — a **NATS Settings** category appears in its details:

| Field | Default | Description |
|---|---|---|
| **Server Address** | `127.0.0.1` | The same as the subsystem's Server URL, but only for this component |
| **Server Port** | `4222` | — |
| **Nats Credentials** | `AuthType = None` | — |
| **Auto Subscribe Subjects** | empty | The same as the settings page, but only for this component |
| **b Auto Reconnect** | `true` | — |
| **Reconnect Delay** | `5.0` | Pause between reconnect attempts, seconds |

The component does **not** read the Project Settings page — every value is configured here
separately, and the defaults differ from the subsystem's (the component's `ReconnectDelay`
is its own independent field, unrelated to the subsystem's `Reconnect Interval Seconds`).

---

## Changing settings at runtime

None of the above requires restarting the game if you change values via code rather than
through Project Settings:

```cpp
const UNatsClientSettings* Settings = GetDefault<UNatsClientSettings>();
// Settings is read-only access to the defaults —
// to change the current connection, call ConnectWithCredentials/SetCredentials,
// as shown above.
```

The Project Settings page itself only sets the **default values** that `Connect To Server`
reads. Calls to `Connect`/`Connect With Credentials` with explicit parameters ignore it
entirely.

---

**Next:** [4. Credentials: How Not to Store Secrets in the Client](04-Credentials-And-Secrets.md)
