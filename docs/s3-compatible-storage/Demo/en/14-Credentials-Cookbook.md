*🇬🇧 English | [🇺🇦 Українська](../uk/14-Credentials-Cookbook.md)*

[← Back to contents](README.md)

# 14. Credentials Cookbook

A hands-on chapter. Not "where to keep keys" and not "which asset to set up" — those are
already answered by [3. Credentials](03-Credentials.md) and
[13. Profiles in Practice](13-Profile-Scenarios.md). This one is about **the sequence of
actions**: what to call, in what order, in C++ and in Blueprint, for each of the four
configurations.

| Chapter | Answers |
|---|---|
| [3. Credentials](03-Credentials.md) | Where keys belong, and where they don't, and how the plugin looks for them |
| [13. Profiles in Practice](13-Profile-Scenarios.md) | Which profile assets to set up for each configuration |
| **14 (this one)** | Exactly which calls to make, and in what order |

Every scenario below gives numbered steps, a C++ snippet, and an equivalent Blueprint graph.

---

## Four ways to hand a client its keys

The plugin has no "field for a key." Keys reach a client through one of four mechanisms, and
choosing between them is a choice of **who knows them, and when**.

| What you have | Blueprint | C++ | What it acts on |
|---|---|---|---|
| Keys the backend just returned, one storage | **Set Runtime Credentials** (on the subsystem) | `UDemoS3Subsystem::SetRuntimeCredentials` | The default client only |
| The same, but there are multiple storages | **Set Static Credentials** (on a client) | `UDemoS3Client::SetStaticCredentials` | Whichever client it's called on — including a profile's client |
| Keys need to be fetched fresh every time they expire | — | `UDemoS3Client::SetCredentialsProvider` with `FDemoS3CredentialsProvider_Callback` | That client |
| The user entered keys that need to survive a restart | **Save S3 Credentials** | `UDemoS3CredentialsLibrary::SaveUserCredentials` | Every client with `Local user store` and this profile name |
| There are no keys and never will be | `Credential Source = Anonymous` | `FDemoS3CredentialsProvider_Anonymous` | The client |

And the reverse actions:

| Task | Blueprint | C++ |
|---|---|---|
| Forget keys set from code | **Clear Runtime Credentials** | `UDemoS3Subsystem::ClearRuntimeCredentials` |
| Forget the user's keys (one profile / all) | **Clear S3 Credentials** / **Clear All S3 Credentials** | `UDemoS3CredentialsLibrary::ClearUserCredentials` / `ClearAllUserCredentials` |
| Force a profile's client to re-read its source | **Forget Profile Client** | `UDemoS3Subsystem::ForgetProfileClient` |
| Resolve keys right now, without passing anything | — | `UDemoS3Client::RefreshCredentials` |

`SetCredentialsProvider` and `RefreshCredentials` are deliberately not exposed to Blueprint: the
first takes a `TSharedRef` to an interface, the second a `TFunction`. Both belong exactly where
code is already being written.

> **`Set Static Credentials` isn't "the same thing, just longer."** The difference from
> `Set Runtime Credentials` is covered in
> [13. Profiles in Practice](13-Profile-Scenarios.md#how-a-profile-gets-its-credentials-four-sources-in-action):
> the second only sees the default client, and silently does nothing for a client from
> `Get S3 Client For Profile`.

---

## Which source for which build

`EDemoS3CredentialSource` has exactly four values (`Core/DemoS3Enums.h`). The table below is about
where each one belongs, and where it's actively harmful.

| Source | A game players download | Listen host | Dedicated server | Desktop application | Editor and CI |
|---|---|---|---|---|---|
| **Environment variables** | ❌ there are no variables there — every request ends in `Authentication Error`; and if you put them there anyway, you've handed the keys to the player | ❌ the host is the exact same binary as the client | ✅ its main purpose | ⚠️ technically works, but the keys end up being yours, not the user's | ✅ substituted by the editor-only section in the editor |
| **Local user store (encrypted)** | ❌ the player has no S3 keys of their own, and there's nothing to ask them for | ❌ same as above | ⚠️ a server usually runs with no interactive user | ✅ its main purpose | ✅ behaves the same as in production |
| **Anonymous** | ✅ for a bucket open for reading only, and for working through presigned URLs | ✅ same as above | ⚠️ a server usually needs write access | ⚠️ only if someone else's storage is public | ✅ |
| **Supplied in code** | ✅ short-lived keys from your backend | ✅ same as above | ✅ if your secrets manager supplies the keys, rather than the environment | ⚠️ the keys still have to be stored somewhere | ✅ |

Three rules that follow from the table and have no exceptions:

1. **Anything inside the build gets extracted.** This isn't caution for its own sake — see
   [«The Main Rule»](03-Credentials.md#the-main-rule).
2. **`Environment variables` is a server-side source.** Not "almost fits" a client — it doesn't
   fit at all.
3. **`Anonymous` against a private bucket gives `Permission Denied`, not
   `Authentication Error`.** From the provider's perspective, an unsigned request isn't "no
   key" — it's `AccessDenied`, and the error points at policy even though you simply never set
   a provider at all. This is exactly why `DemoMakeDefaultCredentialsProvider` deliberately does
   **not** append `Anonymous` to the end of its chain.

---

## A Provider That Fetches Its Own Fresh Credentials

`FDemoS3CredentialsProvider_Callback` (`Auth/DemoS3CredentialsProviders.h`) is exactly what turns "my
backend issues short-lived keys" into a working scheme. It does three things you'd otherwise
have to write yourself:

- **caches** whatever you handed it, and doesn't ask again while it's still valid;
- **refreshes ahead of time**, `RefreshMarginSeconds` seconds before `ExpiresAtUtc` (60 by
  default), so a request signed right at the end doesn't arrive after the lifetime has already
  run out;
- **coalesces concurrent requests**: twenty transfers starting out on expired keys at once give
  your backend **one** call, not twenty.

A complete example — with error handling, a lifetime, and a reaction to a revoked key:

```cpp
#include "DemoS3Subsystem.h"
#include "DemoS3Client.h"
#include "Auth/DemoS3CredentialsProviders.h"

void UMyGameInstance::InstallS3Credentials()
{
    UDemoS3Client* Client = GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

    // The fetch function is called on demand: once at the first request, and again shortly
    // before the cached credentials lapse. Never on a timer of your own.
    auto Provider = MakeShared<FDemoS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
        FDemoS3CredentialsProvider_Callback::FDemoS3CredentialsFetch::CreateWeakLambda(this,
            [this](FDemoS3CredentialsResolved OnFetched)
            {
                MyBackend::RequestS3Credentials(
                    [OnFetched](bool bOk, const FStsResponse& Response)
                    {
                        if (!bOk)
                        {
                            // Answer anyway. The delegate must fire exactly once per call -
                            // swallowing a failure leaves every waiting transfer hanging.
                            OnFetched.ExecuteIfBound(false, FDemoS3Credentials());
                            return;
                        }

                        // All three values, not two: a temporary key without its session
                        // token is rejected as though it did not exist.
                        OnFetched.ExecuteIfBound(true, FDemoS3Credentials(
                            Response.AccessKeyId,
                            Response.SecretAccessKey,
                            Response.SessionToken,
                            Response.ExpiresAtUtc));   // UTC, and it is what drives the refresh
                    });
            }),
        /*InRefreshMarginSeconds=*/90.f);   // raise it if your backend is slow to answer

    Client->SetCredentialsProvider(Provider);
}
```

**What not to do.** Don't build your own refresh timer, and don't call `RefreshCredentials` on
a schedule: the provider already asks exactly when it needs to, and an extra call is just an
extra request to your backend.

**A revoked key.** If the provider answers `SignatureDoesNotMatch`, `InvalidAccessKeyId`,
`ExpiredToken`, `InvalidToken`, `TokenRefreshRequired` or `RequestTimeTooSkewed`, the plugin
itself calls `Invalidate()` on the provider, and the next request goes out for fresh keys. An
ordinary `AccessDenied` deliberately does **not** do this: the key is correct, it just lacks
permission, and asking again would return the same key while loading the backend for nothing.

**What's available from Blueprint.** Nothing — and that's not a limitation, it's the actual
boundary: the provider's parameter is a function, not a value. The Blueprint equivalent of a
callback provider is calling **Set Runtime Credentials** (or **Set Static Credentials**) every
time your backend responds, with `Expires In Seconds` filled in:

```
On Backend Auth Response
   │
   └─► Get S3 Subsystem → Set Runtime Credentials
           Access Key Id      : (from the backend)
           Secret Access Key  : (from the backend)
           Session Token      : (from the backend)
           Expires In Seconds : 3600
```

The one difference: here **you're** responsible for calling the backend again before the
lifetime runs out. A provider would do it for you.

---

## Scenario 1. Single-Player Game: A 15-Puzzle with a Background from Storage

The game is a classic 15-puzzle. The background image, cut into 15 tiles, doesn't ship in the
build: it comes from storage, so the set of images can grow without a patch.

### Steps

1. Put an **ordinary image file** in the bucket — `puzzle/backgrounds/city.jpg`, with
   `Content-Type: image/jpeg`.
2. Read it **into memory**, not into a file: `S3 Download Bytes` /
   `UDemoS3Client::DownloadBytes`. An intermediate file has no role here — the bytes go straight to
   a decoder.
3. Build a `UTexture2D` from the bytes with `FImageUtils::ImportBufferAsTexture2D`
   (`ImageUtils.h`, module `Engine`). In Blueprint, the same thing is done by the
   **Import Buffer as Texture 2D** node (category `Rendering`).
4. **Store a reference to the texture in a `UPROPERTY`** — it's transient, belongs to no
   package, and without a reference the garbage collector takes it, leaving you a black tile.
5. Use it: `Set Texture Parameter Value` on a dynamic material instance for the board, or
   `Set Brush from Texture` on an `Image` widget.

### C++

```cpp
// PuzzleBoard.h
UCLASS()
class APuzzleBoard : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

private:
    void OnBackgroundDownloaded(const FDemoS3OperationResult& Result, const TArray<uint8>& Data);

    /** Keeps the runtime texture alive: it belongs to no package, so nothing else holds it. */
    UPROPERTY(Transient)
    TObjectPtr<UTexture2D> BackgroundTexture;

    UPROPERTY(Transient)
    TObjectPtr<UMaterialInstanceDynamic> BoardMaterial;
};
```

```cpp
// PuzzleBoard.cpp
#include "ImageUtils.h"     // FImageUtils::ImportBufferAsTexture2D
#include "DemoS3Subsystem.h"
#include "DemoS3Client.h"

void APuzzleBoard::BeginPlay()
{
    Super::BeginPlay();

    UDemoS3Client* Client = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

    // Straight into memory: the bytes go to a decoder, never to disk.
    // CreateUObject is safe here - the plugin invokes the delegate with ExecuteIfBound, so
    // an actor destroyed mid-download simply stops being called.
    Client->DownloadBytes(
        TEXT("my-game-content"),
        TEXT("puzzle/backgrounds/city.jpg"),
        FDemoS3OnDownloadResult::CreateUObject(this, &APuzzleBoard::OnBackgroundDownloaded));
}

void APuzzleBoard::OnBackgroundDownloaded(const FDemoS3OperationResult& Result, const TArray<uint8>& Data)
{
    if (!Result.IsSuccess())
    {
        // ToLogString carries status, code, message and the hint about what to change.
        UE_LOG(LogTemp, Warning, TEXT("Background unavailable, using the built-in one: %s"),
            *Result.ToLogString());
        return;
    }

    // Decodes PNG, JPEG, BMP, TGA, EXR, HDR, TIFF - whatever the engine's ImageWrapper
    // module recognises. Returns null when the bytes are not an image at all.
    BackgroundTexture = FImageUtils::ImportBufferAsTexture2D(Data);
    if (!BackgroundTexture)
    {
        UE_LOG(LogTemp, Warning, TEXT("Downloaded %d bytes, but they are not a picture."), Data.Num());
        return;
    }

    BoardMaterial->SetTextureParameterValue(TEXT("Background"), BackgroundTexture);
}
```

Three properties of this path worth knowing ahead of time:

- **The texture is transient and has a single mip level.** For UI, and for a flat 15-puzzle
  board, that's exactly what you want; for an object seen from a distance, it would be missing
  mips and would "shimmer."
- **Decoding is synchronous.** The plugin guarantees the callback arrives on the game thread
  (see [6. C++ API](06-Cpp-API.md)), so unpacking a 4K JPEG is a noticeable frame hitch. Keep
  images in the bucket at the size you actually need, or store several sizes and read the one
  you want.
- **The object's `Content-Type` has no effect here at all** — the format is determined by the
  bytes themselves. Setting it correctly is still worth doing: it's what makes the object open
  properly in a browser and in the provider's console.

### Blueprint

```
Event BeginPlay
   │
   └─► Get S3 Subsystem → Get Default S3 Client
          │
          └─► S3 Download Bytes
                 Bucket Name : my-game-content
                 Object Key  : puzzle/backgrounds/city.jpg
                    │
                    ├─ On Success → Data ──► Import Buffer as Texture 2D
                    │                            │
                    │                            └─► Return Value
                    │                                   │
                    │                                   ├─► SET Background Texture   ← the actor's variable;
                    │                                   │                              without it, GC takes the texture
                    │                                   │
                    │                                   └─► Set Texture Parameter Value
                    │                                          Target         : Board Material
                    │                                          Parameter Name : Background
                    │                                          Value          : Return Value
                    │
                    └─ On Failure → Result → Get S3 Diagnostic Hint → keep the built-in background
```

The `Background Texture` variable isn't there "for convenience" — it's what keeps the texture
alive. Plug `Import Buffer as Texture 2D` straight into `Set Texture Parameter Value` with
nothing stored anywhere, and the picture disappears on the next garbage collection pass.

### The same, for sound: a tile click from storage

Alongside the background, it's natural to pull in the tile set's sound theme too — a tile click,
a victory sound. The scheme is the same: bytes into memory, an object assembled at runtime, a
reference in a `UPROPERTY`. The only difference is what you assemble it with.

Put a **16-bit PCM `.wav`** in the bucket — `puzzle/sfx/tile-move.wav`, `Content-Type:
audio/wav`. The format isn't a whim here: the path below reads a WAV header and hands the
engine ready-made PCM. MP3 and OGG don't work this way — their decoders in the engine are tied
to cooked assets, so a compressed format would need a codec of your own.

```cpp
#include "Audio.h"                       // FWaveModInfo
#include "Sound/SoundWaveProcedural.h"
#include "Kismet/GameplayStatics.h"

// Same UPROPERTY rule as the texture: a runtime-built sound has no package, so without a
// hard reference the collector takes it and the tile clicks in silence.
UPROPERTY(Transient)
TObjectPtr<USoundWaveProcedural> TileClickSound;

void APuzzleBoard::FetchTileClickSound(UDemoS3Client* Client)
{
    TWeakObjectPtr<APuzzleBoard> WeakThis(this);

    Client->DownloadBytes(
        TEXT("puzzle-assets"), TEXT("puzzle/sfx/tile-move.wav"),
        FDemoS3OnDownloadResult::CreateLambda(
            [WeakThis](const FDemoS3OperationResult& Result, const TArray<uint8>& Bytes)
            {
                if (!WeakThis.IsValid() || !Result.IsSuccess())
                {
                    return;
                }

                // Parses the RIFF header in place: WaveInfo points into Bytes rather than
                // copying, so it stays valid only for as long as Bytes does - which is this
                // callback. QueueAudio below takes its own copy, so that is where it ends.
                FWaveModInfo WaveInfo;
                if (!WaveInfo.ReadWaveInfo(Bytes.GetData(), Bytes.Num()))
                {
                    return;   // not a WAV this path can read
                }

                USoundWaveProcedural* Sound = NewObject<USoundWaveProcedural>();
                Sound->SetSampleRate(*WaveInfo.pSamplesPerSec);
                Sound->NumChannels = *WaveInfo.pChannels;
                Sound->SoundGroup  = SOUNDGROUP_Effects;
                Sound->bLooping    = false;

                const int32 BytesPerFrame = *WaveInfo.pChannels * (*WaveInfo.pBitsPerSample / 8);
                Sound->Duration = BytesPerFrame > 0
                    ? (float)WaveInfo.SampleDataSize / (BytesPerFrame * *WaveInfo.pSamplesPerSec)
                    : 0.f;

                // Copies the samples into the queue the audio thread drains.
                Sound->QueueAudio(WaveInfo.SampleDataStart, WaveInfo.SampleDataSize);

                WeakThis->TileClickSound = Sound;
            }));
}
```

From there — an ordinary `UGameplayStatics::PlaySound2D(this, TileClickSound)`.

> **One difference from the texture worth knowing ahead of time.**
> `USoundWaveProcedural` is a **queue**, not a buffer: playing it drains it. For a sound heard
> once per game, that's plenty; for a tile click that plays hundreds of times, keep the
> downloaded bytes in a `TArray<uint8>` and call `QueueAudio` before every playback (or build
> your own `USoundWave` from `RawPCMData` if you need genuine reuse).

There's no ready-made Blueprint node for assembling a sound from bytes — unlike
`Import Buffer as Texture 2D`. So this step has to be wrapped in your own
`UFUNCTION(BlueprintCallable)`, called from the graph exactly where the background example put
`Import Buffer as Texture 2D`:

```
S3 Download Bytes
   Bucket Name : puzzle-assets
   Object Key  : puzzle/sfx/tile-move.wav
   │
   ├─ On Success → Data ──► Make Sound From Wav Bytes   ← your UFUNCTION from the code above
   │                            │
   │                            └─► Return Value → Set (the Tile Click Sound variable)
   │
   └─ On Failure → Result → Get S3 Diagnostic Hint → keep the built-in sound
```

**Nothing** changes about credentials here: it's the same bucket, the same client, and the same
question of "what happens if this key leaks" — just a different object in it.

### And now the important part: where this game's keys actually come from

The game is single-player — but it **ships to players**, and so falls entirely under
[«The Main Rule»](03-Credentials.md#the-main-rule): anything in the build gets extracted. For
this background specifically, that plays out like this:

| Option | In the build | Costs you | When to use it |
|---|---|---|---|
| **A public bucket + `Anonymous`** | nothing | Anyone can download the images, and the bandwidth bill is yours | A background, patches, any read-only content — **this is a normal answer**, not a compromise |
| **Presigned URLs from your backend** | nothing | Needs a backend and a network — an offline game becomes an online one | When the content is paid or personal |
| **Short-lived keys from your backend** | nothing | The same, plus STS on the backend side | When the game needs write access too, not just reads — cloud saves |
| **A hardcoded, read-only key** | a key | A leak costs bandwidth, not data | When there's no backend and the bucket can't be made public |
| ~~`Environment variables`~~ | nothing | Every request becomes `Authentication Error` | Never in a client build |

**A public, read-only bucket is a legitimate answer.** A 15-puzzle background isn't a secret; it
ends up on the player's disk five seconds after the game starts regardless. All you're giving up
is the ability to prevent people from downloading your images without the game, and you pay for
that in bandwidth. Put a CDN in front of the bucket, keep **only** read-only content in it, and
the question is settled.

A one-question check, from [chapter 3](03-Credentials.md#c-public-read-only-content):
**what happens if this key shows up on a forum tomorrow?** For a read-only key on one public
bucket, the answer is "nothing" — and then it's fine to embed it.

The moment cloud saves show up in the same game, that's already a **second** storage with
different permissions and a different key source, not the same one. The breakdown into two
profile assets is in
[13. Profiles in Practice](13-Profile-Scenarios.md#scenario-1-single-player-game).

---

## Scenario 2. Networked Game: Listen Server

The host is simultaneously server and player, and its binary is no different from a client's.
So long-lived keys never belong here, **for anyone** — the host included.

### Option A. The host does everything

World state belongs to the session, not to a player. Only the host gets keys, and only it talks
to storage.

**Steps:**

1. Set up a profile with `Credential Source = Supplied in code` (say `SP_WorldState`).
2. After authenticating **on the host side**, ask the backend for keys with a policy scoped to
   the session's prefix — `sessions/${session_id}/*`.
3. Hand them to **this specific profile's** client through `SetStaticCredentials`.
4. Guard every operation with an authority check, or the event runs everywhere.

```cpp
void AMySessionState::OnSessionCredentialsReceived(const FStsResponse& Response)
{
    // Authority only: on a listen server this code path exists on every machine.
    if (!HasAuthority())
    {
        return;
    }

    UDemoS3Subsystem* Subsystem = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>();
    UDemoS3Client*    Client    = Subsystem->GetClientForProfile(WorldStateProfile);

    // Applies to this client only - unlike SetRuntimeCredentials, which the subsystem
    // applies to the default client and nothing else.
    Client->SetStaticCredentials(
        Response.AccessKeyId,
        Response.SecretAccessKey,
        Response.SessionToken,
        /*ExpiresInSeconds=*/3600);
}
```

```
Event On Session Credentials Received
   │
   └─► Switch Has Authority
          ├─ Authority → Get S3 Subsystem → Get S3 Client For Profile (SP_WorldState)
          │                 │
          │                 └─► Set Static Credentials
          │                        Access Key Id      : (from the backend)
          │                        Secret Access Key  : (from the backend)
          │                        Session Token      : (from the backend)
          │                        Expires In Seconds : 3600
          │
          └─ Remote → do nothing
```

Then the operation itself, also under `Switch Has Authority`:

```
Event Save World State
   │
   └─► Switch Has Authority
          ├─ Authority → Get S3 Client For Profile (SP_WorldState) → S3 Upload File
          │                 ├─ On Success → Multicast: "world saved"
          │                 └─ On Failure → Result → Get S3 Diagnostic Hint
          └─ Remote    → do nothing
```

Without this check, an eight-player session gives eight identical uploads of the same file,
eight billed requests, and an unpredictable write order.

### Option B. Each client has its own short-lived keys

Justified when the data belongs to a specific player: their own screenshots, their own profile,
their own settings. Here, each client talks to storage on its own, with **its own** keys.

**Steps:**

1. The key's policy is limited to this player's own prefix — `users/${user_id}/*`. One shared
   key for everyone means anyone can overwrite anyone else's data.
2. Keys are fetched **by each process on its own**: `UDemoS3Subsystem` is a
   `UGameInstanceSubsystem` — the host has its own, every client has its own, and nothing
   replicates between them
   (see [10. Deployment Scenarios](10-Deployment.md#the-subsystem-and-networking)).
3. A session outlasts the keys' lifetime — so a
   [callback provider](#a-provider-that-fetches-its-own-fresh-credentials) is the right choice
   here, rather than a one-time call: it refreshes the keys mid-match with no code from you at
   all.
4. **Never send keys over the network.** An RPC with a key as a parameter puts the key in
   traffic and in logs; each client should get its own directly from the backend.

```cpp
// On each client, after it has authenticated against your backend itself.
UDemoS3Client* Client = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

Client->SetCredentialsProvider(
    MakeShared<FDemoS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
        FDemoS3CredentialsProvider_Callback::FDemoS3CredentialsFetch::CreateWeakLambda(this,
            [this](FDemoS3CredentialsResolved OnFetched)
            {
                // Scoped to this player's own prefix by the backend, not by the client.
                MyBackend::RequestS3CredentialsForCurrentPlayer(OnFetched);
            })));
```

```
Event On Player Authenticated        (runs separately on each client)
   │
   └─► Get S3 Subsystem → Set Runtime Credentials
           Access Key Id      : (from the backend, for THIS player)
           Secret Access Key  : (from the backend)
           Session Token      : (from the backend)
           Expires In Seconds : 900
   │
   └─► ... then S3 Upload File with the key users/{PlayerId}/screenshot.png
```

From Blueprint, the lifetime has to be refreshed by hand: call the backend again and call
**Set Runtime Credentials** again before `Expires In Seconds` runs out.

---

## Scenario 3. Dedicated Server

The one configuration where long-lived keys belong: the server binary never ends up with
players.

### Steps: environment variables

1. `Credential Source = Environment variables` — in Project Settings or on a profile asset.
2. If there's more than one storage, give each profile its own **Environment Variable Prefix**.
   The field exists **only on a profile asset** — Project Settings has no such field, and always
   reads the standard `AWS` prefix.
3. Set the variables in whatever launches the process: a systemd unit, Docker, a script, an
   instance role, a secrets manager.
4. Don't call anything from code. That's the whole point of this source.

```bash
# Standard names: the same ones aws, mc, rclone and the rest read.
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...        # temporary credentials only

# A second storage, on the profile whose prefix is REPLAYS.
export REPLAYS_ACCESS_KEY_ID=...
export REPLAYS_SECRET_ACCESS_KEY=...

./MyGameServer -log
```

Values are read **before every request**, so rotating keys takes effect without restarting the
process.

The cheapest way to confirm the server actually sees them is the debug console — the
`S3CompatibleStorageDemoDebug` module ships in a packaged Development build too:

```
> DemoS3TestConnection
```

The full command list is in [12. Debug Console](12-Debug-Console.md).

### A client with no keys at all: presigned URLs

The scheme where the client has **nothing** worth extracting.

**Steps:**

1. The client is configured as `Anonymous`.
2. The client sends the server an RPC: "I want to upload my save."
3. The server checks permission — this is the single point where the decision gets made — and
   signs a URL.
4. The server returns the string to the client.
5. The client makes an ordinary HTTP request, using **the exact method** the URL was signed
   for.

```cpp
// Server side.
const FDemoS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
    TEXT("my-game-saves"),
    FString::Printf(TEXT("saves/%s/slot1.sav"), *PlayerId),
    EDemoS3HttpMethod::PUT,
    /*ExpiresInSeconds=*/900);   // fifteen minutes is plenty for one upload

if (Url.IsSuccess())
{
    // Url.ExpiresAt is already computed: UtcNow plus the lifetime, so the client can show
    // "valid for another N minutes" without subtracting anything itself.
    ClientReceiveUploadUrl(Url.Url, Url.ExpiresAt);
}
```

```
Client: RPC to the server — "I want to upload my save"
   │
Server (Switch Has Authority → Authority)
   │
   └─► Get S3 Subsystem → Get Default S3 Client
          │
          └─► Make Presigned URL                      ← a pure node, no execution pins
                 Bucket Name        : my-game-saves
                 Object Key         : saves/{PlayerId}/slot1.sav
                 Method             : PUT
                 Expires In Seconds : 900
                 │
                 └─► Return Value ──► Client RPC: Receive Upload Url
                                          │
                                    Client: an ordinary HTTP PUT to that address
```

Worth remembering about the URLs themselves:

- A URL is only valid **for the method it was signed with**. A PUT URL opened in a browser
  won't work: browsers send GET.
- The maximum per the Signature Version 4 spec is 604800 seconds (7 days). A larger value isn't
  rejected — it's silently clamped to the maximum, with a warning written to the log.
- The URL grants access to **anyone** who has it. Minutes, not a day.
- The signature is computed locally, with no call to the provider — which is why the
  **Make Presigned URL** and **Generate Presigned URL** nodes are pure and have no
  `On Success`. The difference between them is covered in
  [4. Blueprint Operations](04-Blueprint-Operations.md#which-of-the-two-nodes-to-use).

### A trap: a presigned URL on a callback provider

Signing is **synchronous**, while `FDemoS3CredentialsProvider_Callback` may answer asynchronously.
If nothing is cached in the provider yet at the moment `GeneratePresignedUrl` is called, it
won't answer in time, and the result will be `Authentication Error` with an empty `Url` — with
perfectly working keys.

`Static`, `Environment` and `LocalUserStore` all answer immediately, so this doesn't affect
them at all. For a callback provider, warm the cache once:

```cpp
// Once, before the first presigned URL - not before every one: the provider caches.
Client->RefreshCredentials([this, Client, PlayerId](bool bSuccess)
{
    if (!bSuccess)
    {
        return;
    }

    const FDemoS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
        TEXT("my-game-saves"),
        FString::Printf(TEXT("saves/%s/slot1.sav"), *PlayerId),
        EDemoS3HttpMethod::PUT, 900);

    ClientReceiveUploadUrl(Url.Url, Url.ExpiresAt);
});
```

The simplest way to sidestep this entirely: on a dedicated server, use
`Environment variables`. Then the question never comes up in the first place.

> **Don't put a server-side profile in the client build**, hoping "it just won't work there."
> It really won't — but the failure will look like an unexplained `Authentication Error`.

---

## Scenario 4. Desktop Application

The bucket belongs to **the user**. The keys are theirs, not yours: they can't be hardcoded,
and there's no backend to ask for them.

**Asset:** `Credential Source = Local user store (encrypted)`, `Local Store Profile Name` of
your choosing (typically `default`).

### Step 1. The entry screen

Your settings widget has three input fields and a button. The button calls
**Save S3 Credentials**.

```
("Connect" pressed)
   │
   └─► Save S3 Credentials
          Profile Name      : default
          Access Key Id     : (from an input field)
          Secret Access Key : (from an input field)
          Session Token     : (empty, unless the keys are temporary)
             │
             ├─ Return Value = False → the user's settings folder isn't writable;
             │                          show this, since a silent failure here is the worst outcome
             └─ Return Value = True  → on to verification
```

```cpp
#include "Blueprint/DemoS3CredentialsLibrary.h"

const bool bSaved = UDemoS3CredentialsLibrary::SaveUserCredentials(
    TEXT("default"),
    AccessKeyIdField->GetText().ToString(),
    SecretAccessKeyField->GetText().ToString(),
    /*SessionToken=*/FString());

if (!bSaved)
{
    // The only reason this returns false: the user's settings directory is not writable.
    ShowError(NSLOCTEXT("MyApp", "SaveFailed", "Failed to save the credentials."));
    return;
}
```

`Session Token` on the node sits under `AdvancedDisplay` — visible once you expand the arrow at
the bottom of the node. Fill it in only for temporary keys: a temporary key without its own
token is rejected as though it didn't exist.

### Step 2. Reset the cache and verify — right away

This is the step that saves the most support load. The same error half an hour later, during
the first real save, no longer explains anything to the user.

```
Save S3 Credentials (succeeded)
   │
   └─► Forget Profile Client (SP_UserStorage)      ← the client caches whatever keys it already resolved
          │
          └─► Get S3 Client For Profile (SP_UserStorage) → S3 List Objects
                 Bucket Name : (from the user's input field)
                 Prefix      : (empty)
                 Max Keys    : 1
                    ├─ On Success → the main screen
                    └─ On Failure → Result → Get S3 Diagnostic Hint → show the text next to the fields
```

**`Forget Profile Client` is mandatory here.** The keys changed *behind* the source, not
through a node: a client that already resolved them once keeps using the old ones, and the
check would show the old failure. Compare this with `Set Static Credentials`, after which
`Forget Profile Client` isn't needed — it replaces the client's keys immediately.

`S3 List Objects` with `Max Keys = 1` is the cheapest check that still confirms the endpoint,
region, keys and the bucket's existence all at once. One request, one response. The `Max Keys`
and `Continuation Token` pins sit under `AdvancedDisplay`: expand the arrow at the bottom of the
node.

### Step 3. Application startup

```
Event Init
   │
   └─► Has S3 Credentials (default)
          ├─ True  → straight to the main screen
          └─ False → show the connection screen
```

`Has S3 Credentials` checks for presence **without decrypting** the file — so it's cheap and
safe to call on every startup.

### Step 4. Sign out and "forget me"

| Action | Node | C++ |
|---|---|---|
| Sign out of one account | **Clear S3 Credentials** (`default`) | `UDemoS3CredentialsLibrary::ClearUserCredentials` |
| Wipe everything | **Clear All S3 Credentials** | `UDemoS3CredentialsLibrary::ClearAllUserCredentials` |
| Show where the file lives | **Get S3 Credentials File Path** | `UDemoS3CredentialsLibrary::GetCredentialsFilePath` |

After either of the two clearing actions, call **Forget Profile Client** — for the same reason
as in step 2.

**Get S3 Credentials File Path** is worth exposing in the UI as a "show folder" button: an
honest answer to "what do you store about me, and how do I delete it by hand?"

### Step 5. Multiple accounts or multiple applications

- **Multiple accounts in one application** — its own `Local Store Profile Name` for each, and a
  profile asset per account. Identical (empty) names mean both read the same keys.
- **Multiple applications of yours on one machine** — nothing to do: the store is salted with
  the project's name (`FApp::GetProjectName()`), so two different applications never read each
  other's keys even with the same profile name.

### What this encryption gives you, and what it doesn't

The file lives under the current user's own settings folders, outside the project, encrypted
with AES-256, with the encryption key derived from the machine's identifier and the
application's name. This protects against reading the file by eye, against a leak through a
backup, and against moving the file to another machine — and it does **not** protect against
someone already running code on this machine as this same user. A detailed, unvarnished account
is in [3. Credentials](03-Credentials.md#how-this-is-actually-stored).

If you need system-level guarantees — Keychain, DPAPI, libsecret — implement
`IDemoS3CredentialsStore` (`Auth/IDemoS3CredentialsStore.h`) on top of them and plug your own
implementation into `FDemoS3CredentialsProvider_LocalUserStore`. The rest of the plugin doesn't
change.

---

## Reference: where things live

| What | Header |
|---|---|
| `EDemoS3CredentialSource` — the four sources | `Core/DemoS3Enums.h` |
| `FDemoS3Credentials`, `IsTemporary`, `NeedsRefresh` | `Auth/DemoS3Credentials.h` |
| `IDemoS3CredentialsProvider`, `FDemoS3CredentialsResolved` | `Auth/IDemoS3CredentialsProvider.h` |
| `_Static` · `_Anonymous` · `_Environment` · `_LocalUserStore` · `_Callback` · `FDemoS3CredentialsProviderChain` | `Auth/DemoS3CredentialsProviders.h` |
| `DemoMakeDefaultCredentialsProvider`, `DemoMakeCredentialsProviderForSource` | `Auth/DemoS3CredentialsProviders.h` |
| `IDemoS3CredentialsStore`, `FDemoS3LocalCredentialsStore` | `Auth/IDemoS3CredentialsStore.h` |
| `SaveUserCredentials` and the rest of the user-store nodes | `Blueprint/DemoS3CredentialsLibrary.h` |
| `SetRuntimeCredentials`, `ClearRuntimeCredentials`, `ForgetProfileClient` | `DemoS3Subsystem.h` |
| `SetStaticCredentials`, `SetCredentialsProvider`, `RefreshCredentials`, `GeneratePresignedUrl` | `DemoS3Client.h` |
| `MakePresignedUrl` | `DemoS3BlueprintLibrary.h` |

`DemoMakeCredentialsProviderForSource` is the one place that turns "where to get keys from" into an
actual provider. Both the settings page and every profile asset go through it, which is why
they behave consistently; earlier, each had its own copy of this decision, and they'd drifted
apart.

---

## When it doesn't work

| Symptom | Most likely cause | What to do |
|---|---|---|
| `Authentication Error` in the packaged game, everything's fine in the editor | `Environment variables` plus the editor-only section, which doesn't ship | Change the source: `Anonymous`, `Supplied in code`, or `Local user store` |
| `Permission Denied` where you expected `Authentication Error` | Source is `Anonymous` and the bucket is private: an unsigned request is `AccessDenied` to the provider | Set a provider, or deliberately open the bucket for reading |
| Keys were set, the profile doesn't see them | `Set Runtime Credentials` was called instead of `Set Static Credentials` | The first only affects the default client |
| Keys were changed, the old ones still work | The client caches resolved keys | **Forget Profile Client**. Not needed after `Set Static Credentials` |
| An empty string instead of a presigned URL | The callback provider hasn't cached anything yet, and signing is synchronous | Call `RefreshCredentials` once before the first URL |
| Temporary keys are rejected even though the key and secret are correct | The `Session Token` wasn't passed | Pass all three values |
| The backend gets a burst of key requests | Keys are being set by hand on every operation | Use `FDemoS3CredentialsProvider_Callback` — it coalesces concurrent requests into one |
| A texture or sound appeared and vanished a few seconds later | Nothing references it, so GC collected it | A `UPROPERTY` in C++, or a variable in Blueprint |
| `Read Wave Info` returns `false` on a perfectly valid file | It's not a 16-bit PCM `.wav` — this path doesn't decode MP3 or OGG | Put a `.wav` in the bucket, or decode it with your own codec |

The first line of the Test Connection report always names the source the keys came from —
`Environment`, `LocalUserStore(<name>)`, `Static(editor keys for <profile>)`,
`Static(editor settings)`, `Static(supplied in code)`, `Anonymous`. When it's unclear "which key
exactly signed this," that's where to start:
[3. Credentials](03-Credentials.md#how-the-plugin-looks-up-credentials-at-runtime).

---

[← Back to contents](README.md)
