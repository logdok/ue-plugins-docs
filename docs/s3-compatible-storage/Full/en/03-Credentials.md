*🇬🇧 English | [🇺🇦 Українська](../uk/03-Credentials.md)*

[← Back to contents](README.md)

# 3. Credentials

The most important chapter in this documentation. It's easy to get wrong here, and the
consequences are irreversible: a key that makes it into a packaged game should be considered
public forever.

This chapter answers three questions: **where keys must never be kept**, **where to keep them
for each way of supplying them**, and **exactly how the plugin looks for them at runtime**.

---

## The main rule

**Anything inside a client build can be extracted.**

This isn't caution for its own sake — it's a technical fact. To sign a request, a key has to
exist in memory in plain form — so a memory dump or an HTTPS proxy with a substituted
certificate gets it. Encrypting assets, obfuscation, "hide it in a `.pak`" — none of it helps:
the decryption key lives in the same binary you're handing the player.

So the question isn't "how do I hide the key," it's **"how do I make sure it isn't there at
all."**

That's exactly why the plugin never offers a field to type a key into "for good":

| Place | Why there's no field for a key there |
|---|---|
| **Project Settings → S3 Compatible Storage** | Stored in `DefaultGame.ini` — travels into the build and into version control |
| **The S3 Storage Profile asset** | Ships inside the build the same way any asset does |

Both store **where to get** keys, not the keys themselves.

---

## What "credentials" actually means

| Part | Purpose |
|---|---|
| **Access Key Id** | The key's identifier. Not a secret — it goes out in every request in plain text |
| **Secret Access Key** | The actual secret. It signs the request; the key itself is **never sent** |
| **Session Token** | Only exists for temporary keys. Sent as `x-amz-security-token` |

One thing worth knowing about the Session Token: **a temporary key without its token doesn't
work** — the provider answers as though no such key exists. If you got a triple from STS, pass
all three values, not two.

---

## Four ways to supply credentials

### A. Code running on a server

A dedicated Unreal server, a backend service, a build agent, a console utility.

The only case where long-lived keys are safe: the binary never ends up in someone else's hands.
Keys live:

- in the process's environment variables;
- in the instance's own IAM role (EC2, ECS, EKS) — then there are no keys at all, and the SDK
  pulls temporary ones from instance metadata; for this plugin that just means your environment
  sets the variables for you;
- in a secrets manager or a mounted file that your startup script turns into environment
  variables.

**Setting:** Credential Source → `Environment variables`.

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...   # temporary credentials only
```

Values are read **before every request**, so rotating keys doesn't require restarting the
process.

> Variable names start with `AWS_` by convention, not because you need Amazon. These are the
> same names read by `aws`, `mc` ([MinIO](https://min.io/)), `rclone` and every other tool,
> regardless of which service you're actually talking to.

**Multiple storages on one server.** Give each profile its own **Environment Variable Prefix**:
a profile with the prefix `ARCHIVE` reads `ARCHIVE_ACCESS_KEY_ID` and
`ARCHIVE_SECRET_ACCESS_KEY`, without interfering with the main one.

---

### B. Game client plus your own backend

The common case for a game. Two working approaches, and both leave the client with no
long-lived keys at all.

#### B1. Presigned URLs — the client has no keys whatsoever

The client asks your server: "give me a link to upload my save." The server, which does have
keys, signs a URL and hands it back. The client makes an ordinary HTTP request to that URL —
there's nothing to sign, because the signature is already baked into the address.

The safest option: there's simply nothing to extract from the client.

**Client setting:** Credential Source → `Anonymous`.

On the server side, a URL is created like this:

```cpp
FS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
    TEXT("my-bucket"), TEXT("saves/player-42.sav"),
    ES3HttpMethod::PUT, 900);   // valid for 15 minutes
```

Things worth keeping in mind:

- The URL is only valid **for the method it was signed with**. A PUT URL opened in a browser
  won't work — browsers send GET.
- The maximum lifetime by spec is 7 days (604800 seconds).
- The URL grants access to **anyone** who has it. Keep the lifetime short.
- The signature is computed locally, with no call to the provider, so this is fast and can't
  "fail" over the network.

#### B2. Short-lived keys

The client authenticates against your backend (a Steam or Epic ticket, a JWT — whatever you
use), the backend calls AWS STS `AssumeRole` and returns a triple with a lifetime from 15
minutes to 12 hours and a narrow policy — say, only `s3:PutObject` on
`bucket/users/${user_id}/*`.

These keys live **only in memory** and are never written to disk.

**Setting:** Credential Source → `Supplied in code`.

From Blueprint:

```
(after a successful authentication against your backend)
   │
   └─► Get S3 Subsystem → Set Runtime Credentials
           Access Key Id      : (from the backend)
           Secret Access Key  : (from the backend)
           Session Token      : (from the backend)
           Expires In Seconds : 3600
```

The plugin stops using the keys a little before the stated lifetime, so a request signed right
at the end doesn't arrive after it has already expired.

To sign out — **Clear Runtime Credentials**.

> **Going through a profile asset needs a different node.** `Set Runtime Credentials` only
> configures **the default client**. A client obtained through `Get S3 Client For Profile`
> doesn't see it at all: it takes its keys exclusively from its own asset's
> `Credential Source`.
>
> For a profile, call **Set Static Credentials** on the client itself — it takes the same four
> values and applies to the client it's called on:
>
> ```
> Get S3 Client For Profile (SP_PlayerSaves) → Set Static Credentials
> ```
>
> Full examples for every game configuration are in
> [13. Profiles in Practice](13-Profile-Scenarios.md).

From C++ you can go further and set a provider that fetches fresh keys on its own:

```cpp
auto Provider = MakeShared<FS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
    FS3CredentialsProvider_Callback::FS3CredentialsFetch::CreateLambda(
        [](FS3CredentialsResolved OnFetched)
        {
            // Request to your backend; call OnFetched once the response arrives.
            // Exactly once, in every case, including on failure: that's exactly what the
            // first parameter is for. Calling it only on success would leave every transfer
            // waiting on the keys hanging forever.
            MyBackend::RequestS3Credentials(
                [OnFetched](bool bOk, const FString& Key, const FString& Secret,
                            const FString& Token, int32 TtlSeconds)
                {
                    OnFetched.ExecuteIfBound(bOk, FS3Credentials(
                        Key, Secret, Token,
                        FDateTime::UtcNow() + FTimespan::FromSeconds(TtlSeconds)));
                });
        }));

Client->SetCredentialsProvider(Provider);
```

A provider like this refreshes keys before they expire on its own and **coalesces concurrent
requests**: if twenty transfers start out on expired keys at once, your backend gets one
request, not twenty. Without this, a burst of transfers would turn into a burst of requests to
you.

---

### C. Public, read-only content

Distributing patches, assets, updates from a bucket whose contents are already public.

Here it's acceptable to embed a key — but **only** one with a policy that allows nothing beyond
`s3:GetObject` on a single bucket. Leaking a key like this costs you bandwidth, not data.

**Setting:** `Anonymous`, if the bucket is genuinely open for reading, or `Supplied in code`
with a hardcoded, read-only key.

Check yourself with one question: **what happens if this key shows up on a forum tomorrow?** If
the answer is "nothing bad" — it's fine to embed.

---

### D. Single-user application with no server

A desktop tool, an asset browser, anything with a "connect to my storage" screen. The bucket
belongs to **the user themself**, so the keys are theirs, not yours. They can't be embedded,
and there's no backend to ask for them.

**Setting:** Credential Source → `Local user store (encrypted)`.

Your settings screen calls:

```
("Connect" pressed)
   │
   └─► Save S3 Credentials
           Profile Name      : default
           Access Key Id     : (from an input field)
           Secret Access Key : (from an input field)
```

And at application startup:

```
Has S3 Credentials (default)
   ├─ True  → straight to the main screen
   └─ False → show the connection screen
```

To sign out — **Clear S3 Credentials**.

#### How this is actually stored

The file lives under the current user's own settings folders — **outside the project** — and
is encrypted with AES-256. The encryption key is derived from the machine's identifier and your
application's name.

An honest account of what this protects against:

- ✅ against reading the file by eye, against an accidental leak through a backup or a shared
  config;
- ✅ against moving the file to another machine — the encryption key won't reproduce there;
- ❌ it **does not protect** against someone already running code on this machine as this same
  user: the key-derivation logic lives in the binary.

This is the same level of guarantee most desktop applications offer without integrating a
system keychain. Unreal has no built-in credential store, so if you specifically need one,
implement the `IS3CredentialsStore` interface on top of Keychain, DPAPI or libsecret, without
changing the rest of the plugin.

The **Get S3 Credentials File Path** node returns the file's path — handy for a "show where my
settings are stored" action.

**Multiple storages.** Give each its own **Local Store Profile Name**: keys are stored under
that name, and store A never sees store B's keys.

---

## Summary table

| Scenario | In the build | On the user's disk | In memory |
|---|---|---|---|
| A — server | nothing | environment variables or an instance role | yes, long-lived |
| B1 — presigned URLs | nothing | nothing | only the URL |
| B2 — short-lived keys | nothing | nothing | yes, for minutes |
| C — public read | a read-only key, hardcoded | in a `.pak` | yes |
| D — user's application | nothing | an encrypted file | yes |

---

## How the plugin looks up credentials at runtime

The order is fixed, and it's worth knowing — it's exactly what explains why "the wrong key" is
sometimes used.

**1. Keys set in code override everything.** `Set Runtime Credentials` or a custom provider on
the client take the highest priority: they came from your backend and are current.

**2. After that, whatever's chosen in Credential Source applies:**

| Credential Source | Where it comes from |
|---|---|
| `Environment variables` | Environment variables with the matching prefix |
| `Local user store` | The user's encrypted file, by profile name |
| `Anonymous` | Nothing — requests go out unsigned |
| `Supplied in code` | Nothing is set; waits for your own provider |

**3. In the editor — and only in the editor — the `Environment variables` source first checks
keys entered by hand:**

- **the profile's own keys** (the `Editor Access Key Id` / `Editor Secret Access Key` fields on
  the asset) — if they're set;
- otherwise the shared **Project Settings → S3 Credentials (Editor Only)** section — but only
  when the profile reads the standard `AWS` prefix.

This lets you press Test Connection right away, without writing code or entering keys into your
own project settings. Both places are stored in
`Saved/Config/<Platform>/EditorPerProjectUserSettings.ini`: a personal file, excluded from
version control by the project's standard `.gitignore`, and **never shipped**. What's typed
there never travels into a build, and never to a teammate.

> **This is the most common source of confusion with multiple storages.** A profile with no
> keys of its own falls back to the shared pair — so a request to Amazon goes out signed with a
> [MinIO](https://min.io/) key. The provider answers that no such key exists, and the error
> blames the key, even though the problem is the configuration.
>
> The first line of the report always names the source:
>
> ```
> ---- https://s3.us-east-2.amazonaws.com, Static(editor keys for SP_Archive), region us-east-2
> ```
>
> `Static(editor keys for <profile>)` — this profile's own keys. `Static(editor settings)` —
> the shared section. `Environment` — real environment variables. `LocalUserStore(<name>)` —
> the user's file.

**4. The packaged game sees none of point 3** — none of the editor-only sections ship. Only
what's described in points 1–2 applies there.

---

## Profile keys: in the editor and in the packaged game

The most common question about the **S3 Storage Profile** asset goes like this: if I typed my
keys straight into the asset's fields, and the asset ships inside the build, how does the
packaged game ever read them back out? The answer: **it doesn't, and not because it's "not
implemented" — because they're physically not there.**

### Why they can't end up in the build

The three fields — **Editor Access Key Id / Editor Secret Access Key / Editor Session Token** —
are protected by four independent mechanisms. Independent, meaning a mistake in any one of them
doesn't open up the rest:

| Mechanism | What it actually does |
|---|---|
| `#if WITH_EDITORONLY_DATA` | In a build without the editor these fields don't even exist as class members — the compiler strips them out entirely |
| `Transient` | Even in the editor, the engine never serializes them into the `.uasset`. The asset ships — but these fields aren't in it |
| The keys don't live in the asset | They're stored separately, in `Saved/Config/<Platform>/EditorPerProjectUserSettings.ini`, under this asset's name. A personal file, excluded from version control, and **never shipped** |
| `#if WITH_EDITOR` around the read itself | The code that substitutes these keys in place of environment variables is entirely absent from a packaged game — there's nothing to substitute, and nowhere to put it |

The practical consequence you can see with your own eyes: **the fields are empty every time the
asset is loaded fresh** — the engine never persists them. The plugin fills them in itself, from
that ini, right after the asset loads. This isn't a bug, it's exactly the visible sign that
there are no keys in the asset itself.

They're only written back when you change one of these three fields specifically. Editing
anything else on a profile whose fields are currently empty doesn't wipe the stored keys.
Clearing all three fields is precisely how you delete keys from the store.

### What the packaged game reads

The asset carries only connection configuration into the build: provider, endpoint, region,
addressing style and transfer parameters. It has no credentials in it by construction — it
names the **source** to get them from instead:

| Credential Source | In the editor | In the packaged game |
|---|---|---|
| **Environment variables** | The profile's own keys first; if there are none and the prefix is the standard `AWS` — the shared editor section; otherwise real environment variables | Only real environment variables: `<PREFIX>_ACCESS_KEY_ID`, `<PREFIX>_SECRET_ACCESS_KEY`, `<PREFIX>_SESSION_TOKEN` |
| **Local user store** | The same | The same: an encrypted file in the user's folder, by `Local Store Profile Name` |
| **Anonymous** | The same | The same: requests go unsigned |
| **Supplied in code** | The same | The same: nothing is set, your own code sets a provider |

In other words, **only one row of the table** differs — the one where editor keys stand in for
environment variables. Every other source behaves in the editor exactly the way it does in
production, which is exactly why it's worth checking `Local user store` right in the editor:
whatever you see there is what the game will see too.

Environment variables are read **before every request**, not once at process start — so
rotating keys on a server takes effect without a restart.

If a packaged game has no such variables, the operation doesn't "hang silently": the credential
source reports failure, and the operation ends with `Authentication Error`, with the message
naming exactly which source was checked.

> **The editor-key fields only apply when `Credential Source = Environment variables`.** For
> any other source, they stay available for input but are never read: a profile with
> `Local user store` takes its keys from the user's encrypted file, and simply ignores whatever
> was typed into these fields. If Test Connection doesn't see keys you just entered, check this
> field first.

---

## Rotation and revocation

- **Environment variables** are read before every request — a replacement takes effect
  immediately.
- **Short-lived keys** refresh themselves if you set `Expires In Seconds` or installed a
  callback provider.
- **After a failure caused by the key itself** (`SignatureDoesNotMatch`, `InvalidAccessKeyId`,
  `ExpiredToken` — typically 401), the plugin discards its cached keys, and the next attempt
  asks for them again: a revoked or expired key doesn't get replayed in a loop. A plain
  `403 AccessDenied` is different: the key signed the request correctly, it just lacks
  permission, so there's nothing to discard — a new request with the same keys would give the
  same result, and if the keys come from a backend, this would only load it for nothing.
- **User keys** are removed by `Clear S3 Credentials`; every profile at once — `Clear All S3
  Credentials`.

---

## What not to do

| Temptation | Why it's bad | Do this instead |
|---|---|---|
| Type a key into `DefaultGame.ini` "temporarily" | It goes into git and into the build; "temporary" outlives the release | The Editor Only section |
| Put keys in a `DataTable` or an asset | Ships inside the build the same way | Scenario B |
| Write your own key "encryptor" for the game | The decryption key sits right next to it, in the same binary | Scenario B |
| One writable key shared by every player | One leak, and anyone can overwrite anyone else's saves | STS with a policy scoped to `${user_id}` |
| A long lifetime on a presigned URL "just to be safe" | The URL works for anyone who gets hold of it | Minutes, not a day |

---

**Next:** [4. Blueprint Operations](04-Blueprint-Operations.md) ·
see also [10. Deployment Scenarios](10-Deployment.md) and
[13. Profiles in Practice](13-Profile-Scenarios.md), where these scenarios are worked out on
concrete profile assets
