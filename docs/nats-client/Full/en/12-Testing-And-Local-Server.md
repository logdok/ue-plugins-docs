*🇬🇧 English | [🇺🇦 Українська](../uk/12-Testing-And-Local-Server.md)*

[← Back to contents](README.md)

# 12. Testing and the Local Server

---

## The local server bundled with the plugin

The plugin's `Server/nats/` folder has a ready-made set of Docker Compose files — no need to
find an image and figure out a configuration yourself.

```
Plugins/NatsClient/Server/nats/
├── docker-compose.yml
├── nats.conf          ← the active configuration (port 4222, JetStream enabled)
├── sample nats.conf    ← an example with authentication and a cluster, for reference
├── start-nats.sh        ← launch on macOS/Linux
└── start-nats.ps1       ← launch on Windows
```

Launch it:

```bash
cd Plugins/NatsClient/Server/nats
./start-nats.sh        # macOS/Linux
# or
.\start-nats.ps1        # Windows (PowerShell)
```

Both scripts are just `docker-compose up -d`; Docker must already be installed. The server
comes up on `127.0.0.1:4222` — exactly the default address and port in
[Project Settings](03-Configuration.md#connection).

### The active configuration (`nats.conf`)

| Parameter | Value |
|---|---|
| Client port | `4222` |
| Monitoring port | `8222` |
| JetStream | Enabled, `store_dir: /data` (mounted as a Docker volume, data survives a container restart) |
| Message size limit | `1MB` (`max_payload`) |
| Authentication | Disabled (commented out in the file) |

### Checking the server is alive

```bash
curl http://localhost:8222/varz | grep version
curl http://localhost:8222/jsz
```

The second command shows JetStream statistics — if the response has a `"streams"` field,
JetStream is definitely enabled. The same thing without a terminal — the **Test JetStream**
button in [Project Settings](02-Quick-Start.md#step-5-verification-right-from-the-editor).

### Enabling authentication locally

`sample nats.conf` is a ready-made example with three users and subject-level access rights
(separate for "player" and "server"), plus an example cluster section. To use it: copy the
sections you need into `nats.conf` (or replace the file entirely) and restart the
container:

```bash
docker-compose restart
```

Then in-game, set the matching `Auth Type`/`Username`/`Password` in
[3. Configuration](03-Configuration.md#credentials).

### Stopping and cleaning up

```bash
docker-compose down          # stop
docker-compose down -v       # stop and wipe JetStream data
```

---

## Testing your own integration

### Manual verification from a separate terminal

The fastest way to confirm the data on the wire is exactly what you expect — an
independent client, the [NATS CLI](https://github.com/nats-io/natscli), that doesn't depend
on Unreal or the plugin at all:

```bash
# In one terminal — listen to everything
nats sub "game.events.>"

# In another — publish manually
nats pub game.events.test "connectivity check"
```

If the message shows up in the CLI subscription but not in the game, the problem is on the
client side (subject, subscription, handler). If it doesn't show up anywhere, the problem is
in the publish itself or on the server.

### An isolated test actor

For checking a specific scenario, it's convenient to have a separate, minimal actor,
unrelated to the rest of your game logic — subscribe, print everything that arrives, and
easily delete it once you're done checking:

```cpp
// A minimal test actor
UCLASS()
class ANatsSmokeTestActor : public AActor
{
    GENERATED_BODY()

protected:
    virtual void BeginPlay() override
    {
        Super::BeginPlay();

        UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
        Nats->OnMessageReceived.AddDynamic(this, &ANatsSmokeTestActor::HandleMessage);
        Nats->OnConnected.AddDynamic(this, &ANatsSmokeTestActor::HandleConnected);
        Nats->ConnectToServer();
    }

    UFUNCTION()
    void HandleConnected()
    {
        GetGameInstance()->GetSubsystem<UNatsClientSubsystem>()->Subscribe(TEXT(">"));
    }

    UFUNCTION()
    void HandleMessage(const FNatsMessage& Message)
    {
        UE_LOG(LogTemp, Warning, TEXT("[NATS SMOKE TEST] %s: %s"), *Message.Subject, *Message.Data);
    }
};
```

Subscribing to a bare `>` (everything, at any depth) is intentionally broad, for temporary
diagnostics only — for permanent code always subscribe to a specific subject or a narrow
pattern.

### Automated tests

The plugin ships with its own automated test suite on the Unreal Automation framework
(`Plugins/NatsClient/Source/NatsClientTests/`) — you can write regression tests for your
own integration the same way.

Running via the editor: **Window → Test Automation** (the Session Frontend's Automation
tab), filter by `NatsClient`, pick the tests you want, **Start Tests**.

Headless, from the command line:

```bash
UnrealEditor-Cmd "<path>/YourProject.uproject" \
  -ExecCmds="Automation RunTests NatsClient" \
  -TestExit="Automation Test Queue Empty" \
  -unattended -nopause -nullrhi
```

Some tests (everything that directly checks protocol parsing and frame building) don't need
an external server. Tests that verify real JetStream data exchange connect to
`127.0.0.1:4222` — start the local server as described above before running them.

---

**Next:** [13. FAQ](13-FAQ.md)
