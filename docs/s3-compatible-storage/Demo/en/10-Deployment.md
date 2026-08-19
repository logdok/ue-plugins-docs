*🇬🇧 English | [🇺🇦 Українська](../uk/10-Deployment.md)*

[← Back to contents](README.md)

# 10. Deployment Scenarios

The same plugin behaves differently depending on where the code actually runs: in a
single-player game, in a networked session, or on a dedicated server. This chapter is about
**who actually talks to storage** in each case, and what follows from that.

---

## The short answer

| | Who talks to storage | Where the keys come from | The main trap |
|---|---|---|---|
| **Single-player game** | The client itself | Scenario B or D | A key hardcoded into the build |
| **Networked game (listen)** | The host, and only the host | Scenario B | Every client talking on its own and paying its own bandwidth |
| **Dedicated server** | The server | Environment variables | Trying to do the same thing on the client |
| **Desktop application** | The client | Scenario D | Developer keys instead of the user's own |

More on the scenarios themselves — [3. Credentials](03-Credentials.md).

---

## The Subsystem and Networking

`UDemoS3Subsystem` is a `UGameInstanceSubsystem`. The practical consequences:

- It exists **separately in every game instance**: the client has its own, the server has its
  own.
- In Play In Editor with multiple windows, each window has its own subsystem and its own
  client.
- It **replicates nothing**. The plugin isn't a networking system: it makes HTTP requests from
  whichever process called it.

So the question "how does this work in multiplayer" really comes down to **"in which process am
I calling this node."** The plugin doesn't decide that for you — and it shouldn't.

---

## Single-player game

The simplest case: there's one process, and it's the one talking to storage.

Typical uses — cloud saves, downloading user-generated content, uploading screenshots or crash
reports.

**What to do:**

1. Keys follow scenario **B2** (short-lived, from your backend) or **B1** (presigned URLs). If
   there's no backend at all and the bucket belongs to the user, scenario **D**.
2. Call from anywhere: an actor's Blueprint, a widget, the Game Instance.
3. Remember the operation is asynchronous: the game keeps running while it's in flight.

**Example: a cloud save**

```
Event Save Game
   │
   ├─► Save Game to Slot            (an ordinary local save)
   │
   └─► Get S3 Subsystem → Get Default S3 Client → S3 Upload File
           Bucket     : Get Default Bucket
           Object Key : saves/{PlayerId}/slot1.sav
           File Path  : (path to the saved file)
              │
              ├─ On Success → show "saved to the cloud"
              └─ On Failure → keep the local save, try again later
```

The order here isn't arbitrary: save locally first, then upload. If there's no network, the
player doesn't lose progress — it's just not synced yet.

---

## Networked game: listen server

The host is both server and player. Everyone else connects to it.

**The main decision: who talks to storage.**

### Option 1 — the host only (recommended)

The host holds world state, uploads it, and hands the result to everyone else through ordinary
replication.

```
Event Save World State   (runs on the server)
   │
   ├─ Switch Has Authority
   │     ├─ Authority → S3 Upload File → (success) → Multicast: "world saved"
   │     └─ Remote    → do nothing
```

Why: one call instead of `N`, one set of keys instead of `N`, and no need to grant every client
write access.

### Option 2 — each client on its own

Justified when the data belongs to a specific player and concerns no one else: their own
screenshots, their own settings, their own profile.

Then each client should get **its own** short-lived keys with a policy limited to its own prefix
(`users/${user_id}/*`). One shared key for everyone is an invitation to overwrite someone else's
data.

### What not to do

**Don't call the node on both the server and the clients without thinking it through.** A
Blueprint event that runs everywhere gives an eight-player session eight identical uploads of
the same file — eight billed requests, with an unpredictable write order.

Guard operations that should happen once with `Has Authority` (or a `Switch Has Authority`
node).

---

## Dedicated server

The most convenient case from a security standpoint: the server binary never ends up with
players, so long-lived keys are appropriate here.

**What to do:**

1. Credential Source → `Environment variables`.
2. The launch environment sets the keys: process variables, an instance role, a secrets
   manager.
3. Clients **don't talk to storage at all** — or only through presigned URLs the server issues.

```bash
# systemd unit, Docker, a startup script - anything that sets the environment
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
./MyGameServer -log
```

### Quirks of a server build

**The plugin works with no world and no actor.** The delay between retries is driven by
`FTSTicker`, not a world timer, so operations run correctly while loading a map, in a console
utility, and in a commandlet.

**The test module never ships in a server build** — it has type `DeveloperTool` and only builds
for the editor and Development.

**There are no Editor Only sections on a server.** This isn't a limitation, it's a guarantee:
whatever you typed in the editor could never have made it into the build. On a server, only
environment variables or whatever your own code sets are in play.

### Handing out presigned URLs

A typical pattern when a client needs to upload something but shouldn't have any keys at all:

```
Client: RPC to the server — "I want to upload my save"
   │
Server: checks permission, then
   │      Make Presigned URL (PUT, 900 seconds)
   │
   └─► returns the address to the client
         │
Client: an ordinary HTTP PUT to that address
```

The client can be configured as `Anonymous` here — there's nothing for it to sign.

---

## Desktop application

The bucket belongs to the user. Scenario **D**: a key-entry screen, an encrypted local store,
`Local user store`.

Two tips that save time:

- **Give the profile a meaningful name** (`Local Store Profile Name`), not `default`, if you
  have more than one application: that way one won't see another's keys.
- **Verify keys right after they're entered** — add a "Test" button that calls
  `S3 List Objects` with a limit of 1. An error right after entry is clear; the same error half
  an hour later, during the first real save, isn't.

---

## Play In Editor: what's worth knowing

- Every PIE window is a separate game instance, so each has its own client.
- Keys come from the editor-only sections (see [3. Credentials](03-Credentials.md)) — the
  packaged game will behave differently, and that's deliberate.
- Running PIE as a dedicated server gives you the exact same split as in production: this is
  the right place to check `Has Authority`, not after packaging.

> **PIE clients are separate, but the disk is shared.** PIE windows are separate game
> instances, but they live in one process and write to one shared `Saved/`. For transfers this
> has a real consequence: two instances running the **same** graph with the same
> `Local File Path` end up working on one file on disk. This shows up most clearly with
> `S3 Download File Chunked` — both will open the same `<file>.s3part` and write to it in
> turns, corrupting the result; the resume records under `Saved/S3/` are shared too, since
> their identifier is computed from the bucket, key and path, and those are identical in both.
>
> This isn't specific to PIE — two copies of a packaged game launched from the same folder on
> the same machine behave the same way. The fix is the same as in [«What Not to
> Do»](#what-not-to-do): either `Has Authority` wherever a transfer should happen once, or a
> different `Local File Path` for each instance.

---

## Pre-release checklist

- [ ] `DefaultGame.ini` and every profile asset **have no keys**.
- [ ] Credential Source matches how keys are actually supplied (not `Environment variables` in
      a game players download).
- [ ] Operations that should run once are guarded with `Has Authority`.
- [ ] The key's policy is scoped down to the actions and prefix it actually needs.
- [ ] Network errors are handled: the player should see clear behavior with no internet
      connection.
- [ ] Large transfers can be cancelled, and cancellation has been tested —
      see [5. Transfers](05-Transfers.md).
- [ ] Keys used during development have been reissued before release.
- [ ] `S3.Live.*` has been run against your own bucket and provider, not just the default
      settings — see [9. Testing](09-Testing.md#live-testing-in-your-own-project).

---

**Next:** [11. FAQ](11-FAQ.md) ·
see also [13. Profiles in Practice](13-Profile-Scenarios.md), where these same configurations
are shown through profile assets and credential sources
