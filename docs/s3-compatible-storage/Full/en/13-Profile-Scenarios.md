*🇬🇧 English | [🇺🇦 Українська](../uk/13-Profile-Scenarios.md)*

[← Back to contents](README.md)

# 13. Profiles in Practice: Game Scenarios

A chapter about how **S3 Storage Profile** assets map onto real project configurations —
single-player, a listen server, a dedicated server, a desktop application — and where each
one's keys come from.

Earlier chapters cover these things separately: [2. Configuration](02-Configuration.md) — what
a profile is, [3. Credentials](03-Credentials.md) — where to keep keys,
[10. Deployment Scenarios](10-Deployment.md) — who talks to storage. Here's where they meet, on
concrete assets with concrete values.

---

## The main rule

**A profile stores not the keys, but the source to get them from.** So "which profile do I
build" always comes down to **"which process will use it"** — because the source has to be
reachable by exactly that process.

Which leads to the consequence that saves the most time:

> A profile with `Environment variables` is a **server-side** profile. If it ships to players
> and they call it, there will be no environment variables there, and every request ends in
> `Authentication Error`. This isn't a plugin bug — it's a source picked for the wrong process.

If both the client and the server work with the same bucket but with different permissions,
that's **two profiles** with the same endpoint and bucket and different sources — not one
shared one.

---

## Do you even need a profile

| Situation | What to use |
|---|---|
| One storage for the whole project | **Project Settings** and `Get Default S3 Client`. No profile needed |
| Two or more storages | A profile for each |
| Configuration known only at runtime (came from a backend) | `Get Named S3 Client` — a profile won't help here, since the asset is fixed at development time |

Profiles start paying off from the second storage onward: the shared pair of editor keys is one
per project and can only describe one of them — see
[«Profile keys»](03-Credentials.md#profile-keys-in-the-editor-and-in-the-packaged-game).

---

## How a Profile Gets Its Credentials: Four Sources in Action

| Credential Source | Who supplies the keys | Available from Blueprint | Typical process |
|---|---|---|---|
| **Environment variables** | The process's own launch environment | — (nothing to call) | Dedicated server, backend, build agent |
| **Local user store** | The end user, through `Save S3 Credentials` | ✅ yes | Desktop application |
| **Anonymous** | No one, requests go unsigned | — | Public bucket, presigned URLs |
| **Supplied in code** | Your own code, with `Set Static Credentials` on the client | ✅ yes | A game client with short-lived keys from a backend |

> **Two similar nodes that act on different clients — the most important thing in this
> chapter.**
>
> | Node | What it acts on |
> |---|---|
> | **Set Runtime Credentials** (on the subsystem) | Only the default client |
> | **Set Static Credentials** (on a client) | Whichever client it's called on — including a profile's client |
>
> Clients from `Get S3 Client For Profile` take their keys exclusively from their own asset's
> `Credential Source`, and they **don't see** `Set Runtime Credentials` at all. This is a quiet
> failure: the node itself doesn't error, the profile just behaves as though it was never given
> keys. A profile needs `Set Static Credentials` specifically.

For a profile with `Supplied in code`, keys are set like this:

```
On Login Response                        (your backend returned short-lived keys)
   │
   └─► Get S3 Subsystem → Get S3 Client For Profile (SP_PlayerSaves)
          │
          └─► Set Static Credentials
                 Access Key Id      : (from the backend)
                 Secret Access Key  : (from the backend)
                 Session Token      : (from the backend)
                 Expires In Seconds : 3600
```

The keys stay **only in memory**: nothing is written to disk, nothing ends up in the build. The
plugin stops using them a little before the stated lifetime, so a request signed right at the
end doesn't arrive after it has already expired. `Expires In Seconds = 0` means keys with no
expiration.

From C++ there's also a provider that fetches fresh keys on its own as old ones expire:

```cpp
US3Subsystem* Subsystem = GetGameInstance()->GetSubsystem<US3Subsystem>();
US3Client*    Client    = Subsystem->GetClientForProfile(ArchiveProfile);

Client->SetCredentialsProvider(
    MakeShared<FS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(/* your request to the backend */));
```

---

## Scenario 1. Single-Player Game

Cloud saves plus public content — two storages with different permissions.

**Assets:**

| Asset | Provider / bucket | Credential Source | Why |
|---|---|---|---|
| `SP_PlayerSaves` | your main one, `my-game-saves` | `Supplied in code` | The player's save: written privately |
| `SP_PublicContent` | the same or a different one, `my-game-content` | `Anonymous` | Patches and assets: read-only, a public bucket |

**Graph after the player logs in** (the backend handed back short-lived keys):

```
On Login Response
   │
   └─► Get S3 Subsystem → Get S3 Client For Profile (SP_PlayerSaves)
          │
          └─► Set Static Credentials
                 Access Key Id      : (from the backend)
                 Secret Access Key  : (from the backend)
                 Session Token      : (from the backend)
                 Expires In Seconds : 3600
```

**From there — an ordinary save:**

```
Event Save Game
   │
   ├─► Save Game to Slot                        (locally first)
   │
   └─► Get S3 Subsystem → Get S3 Client For Profile (SP_PlayerSaves) → S3 Upload File
              │
              ├─ On Success → "saved to the cloud"
              └─ On Failure → keep the local save, try again later
```

Public content needs nothing extra:

```
Get S3 Subsystem → Get S3 Client For Profile (SP_PublicContent) → S3 Download File
```

**Why this shape.** `Anonymous` for the public bucket means the build carries absolutely nothing
extractable. And `Supplied in code` for saves means the keys live only in the process's
memory: never written to disk, never in the build, and gone the moment the player signs out.

> **Don't confuse this with `Local user store`.** That one deliberately **persists** keys to
> disk (encrypted) — correct for a desktop application where the keys belong to the user and
> should survive a restart, and unnecessary for a game where the backend hands out fresh ones
> every time.

`Set Static Credentials` takes effect on the client immediately — **Forget Profile Client**
isn't needed here. It's needed in a different case: when the keys change *behind* the source
(in the local user store, say), because then the client keeps using what it already resolved.

---

## Scenario 2. Networked Game: Listen Server

The host is both server and player. The main decision is **who talks to storage**.

**Assets:**

| Asset | Credential Source | Who uses it |
|---|---|---|
| `SP_WorldState` | `Supplied in code` — the host gets keys from your backend | The host only |
| `SP_PlayerUploads` | `Supplied in code` — each player has their own keys | Each client, on its own |

**World state — host only:**

```
Event Save World State
   │
   └─► Switch Has Authority
          ├─ Authority → Get S3 Client For Profile (SP_WorldState) → S3 Upload File
          │                 └─ On Success → Multicast: "world saved"
          └─ Remote    → do nothing
```

Without `Switch Has Authority`, the event runs everywhere: an eight-player session gives eight
identical uploads of the same file, eight billed requests, and an unpredictable write order.

**A player's own data — each on their own.** Here, each client needs to get **its own** keys,
with a policy scoped to its own prefix (`users/${user_id}/*`). One shared key for everyone is
an invitation to overwrite someone else's data.

---

## Scenario 3. Dedicated Server

The case profiles are really worth setting up for: the server binary never ends up with
players, so long-lived keys are appropriate here, and **`Environment Variable Prefix` gives you
multiple storages in one environment**.

**Assets:**

| Asset | Credential Source | Environment Variable Prefix | Reads |
|---|---|---|---|
| `SP_Saves` | `Environment variables` | `SAVES` | `SAVES_ACCESS_KEY_ID`, `SAVES_SECRET_ACCESS_KEY` |
| `SP_Replays` | `Environment variables` | `REPLAYS` | `REPLAYS_ACCESS_KEY_ID`, `REPLAYS_SECRET_ACCESS_KEY` |

**Launching the server:**

```bash
# systemd unit, Docker, a startup script - anything that sets the environment
export SAVES_ACCESS_KEY_ID=...
export SAVES_SECRET_ACCESS_KEY=...

export REPLAYS_ACCESS_KEY_ID=...
export REPLAYS_SECRET_ACCESS_KEY=...

./MyGameServer -log
```

Two storages, two key pairs, each with its own policy — and not a line of code has to tell them
apart: the profile already knows which prefix to read.

Values are read **before every request**, so rotating keys takes effect without restarting the
process.

**And on the clients.** They don't talk to storage at all — or only through presigned URLs the
server hands out:

```
Client: RPC to the server — "I want to upload my save"
   │
Server: checks permission, then Make Presigned URL (PUT, 900 seconds)
   │
   └─► returns the address to the client
         │
Client: an ordinary HTTP PUT to that address
```

The client is configured as `Anonymous` here — there's nothing for it to sign.

> **Don't put `SP_Saves` into the client build** hoping "it just won't work there." It really
> won't — but the failure will look like an unexplained `Authentication Error`. If a client
> needs access to that same bucket, set up a separate profile with `Anonymous` or
> `Local user store` instead.

---

## Scenario 4. Desktop Application

The bucket belongs to **the user**, so the keys are theirs, not yours.

**Asset:** `SP_UserStorage`, `Credential Source = Local user store`,
`Local Store Profile Name = default`.

```
("Connect" pressed on the settings screen)
   │
   └─► Save S3 Credentials (Profile Name: default, keys from the input fields)
          │
          └─► Forget Profile Client (SP_UserStorage)
                 │
                 └─► Get S3 Client For Profile (SP_UserStorage) → S3 List Objects (Max Keys 1)
                        ├─ On Success → the main screen
                        └─ On Failure → show the error right away, while the user's still there
```

Verifying right after entry is what saves the most support load: the same error half an hour
later, during the first real save, no longer explains anything to the user.

**Multiple applications on one machine** — give each its own `Local Store Profile Name`. The
store is additionally salted with the project's name, so two different applications never read
each other's keys even with the same profile name.

---

## Summary table

| Scenario | Profile | Credential Source | Where the keys actually live | Blueprint? |
|---|---|---|---|---|
| Single-player: saves | `SP_PlayerSaves` | Supplied in code | Memory only, `Set Static Credentials` | ✅ |
| Single-player: public content | `SP_PublicContent` | Anonymous | Nowhere — nothing is signed | ✅ |
| Listen: world state | `SP_WorldState` | Supplied in code | Memory only, on the host | ✅ |
| Dedicated server | `SP_Saves`, `SP_Replays` | Environment variables | The process environment, its own prefix per profile | ✅ |
| Client on a dedicated server | `SP_PublicRead` | Anonymous | Nowhere; access through presigned URLs | ✅ |
| Desktop application | `SP_UserStorage` | Local user store | An encrypted file in the user's folder | ✅ |

---

## Common mistakes

| Mistake | What you see | What to do |
|---|---|---|
| `Environment variables` in a build for players | `Authentication Error` on every operation | This is a server-side source. Use `Anonymous`, `Local user store`, or presigned URLs for the client |
| `Set Runtime Credentials` together with a profile | The profile behaves as though it was never given keys | This node only affects the default client. For a profile — `Set Static Credentials` on its own client |
| Keys changed in the local store, but the old ones still work | The old 403 keeps repeating | `Forget Profile Client` — the client caches what it already resolved. Not needed after `Set Static Credentials`: it replaces the keys immediately |
| Two profiles, keys only in the shared editor section | A request to one storage goes out signed with the other's key | Fill in **Editor Access Key Id** on the profile itself — see [3. Credentials](03-Credentials.md#profile-keys-in-the-editor-and-in-the-packaged-game) |
| An empty `Local Store Profile Name` on two profiles | Both read the same keys | Give each its own name |

---

[← Back to contents](README.md)

**Next:** [14. Credentials Cookbook](14-Credentials-Cookbook.md)
