*🇬🇧 English | [🇺🇦 Українська](../uk/04-Credentials-And-Secrets.md)*

[← Back to contents](README.md)

# 4. Credentials: How Not to Store Secrets in the Client

[3. Configuration](03-Configuration.md#credentials) shows **where** to technically enter
credentials. This chapter is about a different question: **where they don't belong at all**,
and what to do instead. Read it before typing a real password or token into any field the
plugin exposes.

---

## Why this matters at all

Any game is a program that runs on a player's device. Everything that goes into a packaged
game — code, assets, config files — physically sits on the player's disk and can eventually
be read: with `.pak` unpacking tools, a debugger, or simply a text editor if the config isn't
packed. Encrypting assets doesn't save you here: to connect to a server, the client itself
must hold the secret in plain form in memory at the moment of connecting — meaning that,
sooner or later, it's accessible to anyone who fully controls their own device (that is, any
player).

**A practical rule:** if a secret physically made it into a packaged game, consider it
public. The question isn't "can they find it," it's "when." So real protection isn't hiding
the secret better — it's not putting it into what ships to the player at all.

---

## Three places for credentials — and when each is appropriate

| Place | Ends up in the packaged game? | When appropriate |
|---|---|---|
| **Project Settings → `Credentials`** ([3. Configuration](03-Configuration.md#credentials)) | **Yes**, in plain form, in `DefaultGame.ini` | Local development; a server with no sensitive data at all; low-impact leaks (e.g. a shared test token on a closed testbed) |
| **Editor Only credentials** ([below](#editor-only-credentials-for-testing-in-the-editor)) | **No** — physically absent from the build | Testing in the editor (Play In Editor, the Test Connection/Test JetStream buttons) with real, sensitive credentials — without the risk of accidentally committing them into Project Settings |
| **Fetching at runtime** ([below](#runtime-fetching-credentials-from-outside)) | **No** — the secret isn't in any project file at all | Production: the player's game or a dedicated server needs to connect to a real server with real access rights |

The first two only solve the problem for **a developer working in the editor**. If your
**game** (not the editor) connects to a production server, you need the third option,
covered in detail below.

---

## Editor Only credentials: for testing in the editor

**Editor Preferences → Plugins → NATS Messaging Client (Editor Only)**

A separate settings page next to the main one — specifically so you can test in the editor
with real credentials without ever typing them into a field that gets stored in
`DefaultGame.ini`.

### How to enable it

1. Open this page in Project Settings.
2. Turn on **b Use Editor Credentials**.
3. Fill in **Credentials** exactly as on the main page — the same choice of `Auth Type` and
   the same fields (Basic/Token/JWT).

Done. Now:

- **Connect To Server** on the subsystem (both from Blueprint and C++) uses exactly this
  data;
- the **Test Connection** button too;
- the **Test JetStream** button too.

Each of the two buttons appends an `(Editor-only credentials)` tag to its notification popup,
so it's clear which data just ran — the main page's, or the editor override.

### Why this is safe

The values are stored in `Saved/Config/<Platform>/EditorPerProjectUserSettings.ini`, a file
that:

- lives inside the `Saved/` folder, which a standard Unreal project's `.gitignore` excludes
  from version control **by default** — the secret won't reach git even if you forget about
  it;
- belongs to your specific machine and your specific OS user account — it doesn't sync along
  with the project to another computer;
- **is physically absent from the packaged game.** The settings class itself is compiled
  only for editor builds — in Development/Shipping game configurations this data isn't just
  unread, it's simply not present in the binary at all.

### An important limitation

Editor Only credentials only affect **what happens in the editor**. That is:

- `Connect To Server` on the subsystem, when you play in PIE;
- the two test buttons.

It does **not** affect:

- `NATS Client Component` — it has its own `Nats Credentials` field on each actor; the
  editor override doesn't substitute for it;
- the packaged game a player launches — there this settings class doesn't exist at all (as
  just explained above), so there's simply nothing to override.

**In other words: Editor Only credentials solve the testing problem, not the production
problem.** A player's game will still connect with whatever credentials are entered on the
main Project Settings page — meaning that if you left nothing sensitive there, the game
connects with `Auth Type = None` or whatever you pass in via code. That's exactly what the
next chapter is for: connecting a player with real access rights.

---

## Runtime: fetching credentials from outside

This is how a production game or a dedicated server obtains real, sensitive credentials
without storing them in any project file at all. The secret only lives in memory, for the
duration of one session, and gets there from a source you control yourself — not from an
`.ini` file that ships with the build.

The mechanics on the plugin's side are the same regardless of the source: build an
`FNatsCredentials` (or the Blueprint `Make Nats Credentials` struct) and pass it to
`Connect With Credentials` — covered in detail in
[3. Configuration → Where to set credentials](03-Configuration.md#where-to-set-credentials).
The only difference is **where** the values for this structure come from. Below are
concrete, typical sources.

### Scenario 1: the player's game gets a token from your backend after login

The most common case. The player already has an account in **your own** system (not NATS) —
login, registration, or platform authorization (Steam, PlayStation Network, etc.). Your
backend server, once it's confirmed who this player is, issues them a short-lived NATS token
(or a JWT) specifically for this session — and that's what gets passed into the plugin.

```
[Player logged in — Login succeeded]
   │
   └─► [Your HTTP request to the backend: "issue me a NATS token for this session"]
          │
          └─► [The backend's response contains the token]
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

```cpp
void AMyPlayerController::OnBackendLoginSuccess(const FString& NatsToken)
{
    FNatsCredentials Credentials;
    Credentials.AuthType = ENatsAuthType::Token;
    Credentials.Token = NatsToken;

    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->ConnectWithCredentials(TEXT("nats.mygame.com"), 4222, Credentials);
}
```

The plugin doesn't implement the HTTP request to the backend itself — it's an ordinary call
to your own API, using whatever mechanism your project already talks to your backend with
(Unreal's `HTTP` module, or any other tool you use for your own backend protocol).

**Why this is safe.** The token is never written to the player's disk in any config file —
it only exists in the current game session's memory. If a player tries to peek at "the NATS
password" in the game's files, there's simply nothing there. On top of that, the token can
be made short-lived on the backend side (say, valid for one hour) — even if it were somehow
intercepted from traffic or process memory, the damage is time-limited.

### Scenario 2: a dedicated server gets its secret at launch

For a `dedicated server` (where there's no player and no "login" step), the typical source
is the launch environment itself, not a backend request: a command-line argument or an
environment variable, set by whoever deploys the server (you yourself, your DevOps, or an
orchestrator like Kubernetes or Docker Compose with secrets).

```cpp
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    // From a command-line argument: -NatsToken=secret
    FString Token;
    if (!FParse::Value(FCommandLine::Get(), TEXT("NatsToken="), Token))
    {
        // Or from an environment variable, if the command line doesn't have it
        Token = FPlatformMisc::GetEnvironmentVariable(TEXT("NATS_TOKEN"));
    }

    if (Token.IsEmpty())
    {
        UE_LOG(LogTemp, Error, TEXT("No NATS token set: server was launched without -NatsToken and without NATS_TOKEN"));
        return;
    }

    FNatsCredentials Credentials;
    Credentials.AuthType = ENatsAuthType::Token;
    Credentials.Token = Token;

    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->ConnectWithCredentials(TEXT("nats.mygame.com"), 4222, Credentials);
}
```

Launching the server then looks like this:

```bash
./MyGameServer -NatsToken=real_server_secret
# or
NATS_TOKEN=real_server_secret ./MyGameServer
```

> Reading command-line arguments and environment variables (`FParse::Value`,
> `FPlatformMisc::GetEnvironmentVariable`) is a standard Unreal capability, available only
> from C++. There's no ready-made Blueprint node for this in the base engine — if you
> specifically need Blueprint, wrap the call in your own simple `BlueprintCallable` function
> on your project's side.

**Why this is safe.** The secret doesn't live in any game or project file — it lives in the
process environment (an environment variable or a launch argument), controlled by whoever
deploys the server, and typically never enters version control alongside the game code.

### Other sources — briefly

- **A server configured to hand out credentials itself** (an auth callout or similar
  mechanism) — the plugin supports passing a JWT token as-is, without an NKey signature; more
  detail and caveats — [3. Configuration → Credentials](03-Configuration.md#credentials).
- **A system secrets store (Keychain, Windows Credential Manager, etc.).** The plugin doesn't
  provide a ready-made integration — if you need exactly this level of protection, read the
  secret from there with your own code (outside the plugin) and pass the result the same way
  to `Connect With Credentials`. The plugin itself doesn't dictate where the token or
  password string came from — it only accepts the finished result.

---

## Summary: what to choose

```
Is this a production server that a real game or a real player will connect to?
   │
   ├─ No, it's just my local test / a server with no sensitive data
   │      → Project Settings, the main page. Simple and sufficient.
   │
   └─ Yes
          │
          ├─ I'm testing this production server from the editor right now
          │      → Editor Only credentials (this chapter, above)
          │
          └─ The player's actual game or a dedicated server will connect this way
                 → Runtime fetching (this chapter, "Scenario 1" or "Scenario 2" above)
                    + Connect With Credentials
```

---

**Next:** [5. Core Messaging](05-Core-Messaging.md)
