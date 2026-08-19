*🇬🇧 English | [🇺🇦 Українська](../uk/04-Blueprint-Operations.md)*

[← Back to contents](README.md)

# 4. Blueprint Operations

---

## The shape every node shares

Every operation is a latent node with the same set of pins. Learn one, and you know the rest.

| Pin | When it fires |
|---|---|
| **On Started** | Immediately, before any data moves. Gives you a `Transfer` for cancellation and progress |
| **On Progress** | As bytes move, wherever the operation can report it |
| **On Success** | Once, with the result |
| **On Failure** | Once, with the cause and a hint about what to change |

Exactly one of `On Success` and `On Failure` fires **always** — so cleanup is written in one
place, not twice. All pins fire on the game thread.

> **`Transfer` is present on all four pins, not just `On Started`.** It's the same handle every
> time — grabbing it once, on `On Started`, into a variable is enough; on
> `On Progress`/`On Success`/`On Failure` it's simply available again, no need to re-save it.
> `Progress`/`Result`/`Data`, on the other hand, only make sense on their "own" pin: for
> instance, `Progress` on `On Started` or on `On Success` is always empty — that's not a bug,
> there's just nothing to report there.

---

## Where to get a client

The client is the first pin on every operation, and there are three ways to get one. The
difference is only in where the configuration comes from.

### One storage: the default client

```
Get S3 Subsystem → Get Default S3 Client
```

The subsystem creates a client from the project's settings, keeps it alive, and returns the
same instance. No variable is needed — the garbage collector won't touch it.

### Multiple storages: a client from a profile asset

When there's more than one storage, it's convenient to keep each one's configuration in an
**S3 Storage Profile** asset (how to create one — [2. Configuration](02-Configuration.md#multiple-storages-profiles-as-assets)).
To work with a storage like that, exactly one node changes at the start of the chain:

```
Get S3 Subsystem
   │
   └─► Get S3 Client For Profile
          Profile : SP_Archive               ← your S3 Storage Profile asset
          │
          └─► Return Value ──► Client (operation pin)
                                  │
                                  S3 Download File
                                     Bucket Name     : my-game-assets
                                     Object Key      : videos/intro.mp4
                                     Local File Path : Saved/Downloads/intro.mp4
                                        │
                                        ├─ On Started  → Transfer  → Set (a variable, for Cancel)
                                        ├─ On Progress → Progress  → Break S3 Transfer Progress
                                        ├─ On Success  → done, the file is on disk
                                        └─ On Failure  → Result → Get S3 Diagnostic Hint
```

**Where to find the node.** Right-click in the graph → category **S3** → **Get S3 Client For
Profile**. The `Profile` pin is an ordinary object reference: pick the asset from the dropdown
right on the node, or plug in a variable of type `S3 Storage Profile`.

Three things worth knowing right away:

| | |
|---|---|
| **You don't need to store the client** | It's created on first access and reused after that, so calling this node from different places in the graph is cheap |
| **`Bucket Name` stays on the operation itself** | The profile sets the endpoint, region, addressing style and credential source; the bucket is a parameter of the specific operation |
| **Keys come only from the profile's `Credential Source`** | If that's `Supplied in code`, then `Set Runtime Credentials` **won't help** this client — it only configures the default client. Take the profile's client and call **Set Static Credentials** on it |

Full examples for every game configuration are in
[13. Profiles in Practice](13-Profile-Scenarios.md).

### Configuration known only at runtime

If the endpoint and keys arrive from outside during gameplay — from your backend, say — the
asset won't help, because it's fixed at development time. In that case, assemble the
configuration with the **Make S3 Config (Provider)** node and get a client through
**Get Named S3 Client**.

```
[received an endpoint and keys from the backend]
   │
   └─► Make S3 Config (Provider)
          Provider : Custom
          Endpoint : (from the backend's response)
          ...
          │
          └─► Get S3 Subsystem → Get Named S3 Client
                                     Client Name : "BackendStorage"
                                     Config      : Return Value (from above)
                                     │
                                     └─► Client (operation pin)
```

**What `Client Name` actually is.** It's not an identifier issued by S3, by the backend, or by
anyone else — it's a name **you make up yourself**, just to label the client in the
subsystem's cache. The subsystem itself knows nothing about it beyond that it's a key in a
"name → client" map. `"BackendStorage"` or `"MyLittleStorage"` — it doesn't matter which; what
matters is **using the exact same string every time** you want the same client. It's closer to
a variable name than to a setting.

Which leads to a practical habit:

- Pick one stable name per distinct storage the game talks to (if there's only one storage, a
  literal `"Default"` or the project's name works fine). A typo or a different case in the
  string, and the node will quietly create a **second, independent** client that isn't
  configured yet, instead of returning the one you already have.
- Call the node with `Config` only where the configuration was just assembled (as in the
  diagram above) — usually that's once, right after the backend responds. Elsewhere in the
  graph, where a client under that name should already exist, `Config` matters less: as the
  table in [2. Configuration](02-Configuration.md#profiles-vs-get-named-s3-client) points out,
  it's only honored on the first access under a given name, and silently ignored after that —
  so you can pass `Make S3 Config (Provider)` with empty fields, or the same value as the first
  time; it makes no difference anymore.
- It's a plain `FName`, so storing it in a Blueprint variable or even in a Data Table (when
  there are several storages and their keys arrive from outside) works exactly like any other
  string.

**How to get the same client elsewhere.** The client lives in the subsystem's cache, not in
your variable — so carrying it through an `Event Dispatcher` or saving it somewhere special
isn't necessary. But for "get what's already been created" there's a separate node —
**Find Named S3 Client** — which, unlike `Get Named S3 Client`, doesn't ask for `Config` at
all, only a name:

```
[somewhere else in the game — say, a "Download" button on an inventory screen]
   │
   └─► Get S3 Subsystem → Find Named S3 Client
                              Client Name : "BackendStorage"      ← the same string used at creation
                              │
                              └─► Return Value ──► Client (S3 Download File's pin)
```

`Find Named S3 Client` is a pure node (no execution pin): wire up `Client Name` and use
`Return Value` — that's all it needs. If nothing has been created under that name yet (for
instance, the graph ran before `Get Named S3 Client` on `BeginPlay`), it returns `Null` — not
an error, just a signal that the creator has to run first.

**Why not simply call `Get Named S3 Client` a second time.** Formally you can: it also returns
the already-existing client, and `Config` is ignored on a repeated call the same way (the table
in [2. Configuration](02-Configuration.md#profiles-vs-get-named-s3-client)). But then you're
stuck wiring up a spare `Make S3 Config (Provider)` just to fill a required pin — and it's not
obvious why it's there at all if it's not actually configuring anything. `Find Named S3 Client`
exists precisely for this second case: fetch it — with no configuration involved.

---

## Uploading

### File Path: Absolute or Relative

`Local File Path` — on both upload and download — accepts either.

**An absolute path** — a full path from the drive root. Always works unambiguously; the
platform only affects its shape:

| Platform | Example |
|---|---|
| Windows | `C:/Users/YourName/Desktop/save.png` |
| macOS | `/Users/YourName/Desktop/save.png` |
| Linux | `/home/YourName/Desktop/save.png` |

**A relative path** — a short string with no drive letter and no leading slash, for example
`Saved/Screenshots/shot.png`. The plugin expands it against the **project root**
(`FPaths::ProjectDir()`) — the same folder that holds `Content`, `Config` and `Saved`. This
works identically in the editor and in a packaged game, on any platform: the same result, the
same string, expanded to the same absolute path every time.

A few ready-made relative-path examples:

| Relative path | Where it points |
|---|---|
| `Saved/Screenshots/shot.png` | The `Saved/Screenshots` folder under the project root |
| `Saved/SaveGames/slot1.sav` | The same place `Save Game to Slot` writes to |
| `Content/Data/manifest.json` | An ordinary file inside the `Content` folder, not an asset |

**What doesn't work.** An asset path like `/Game/Textures/Icon` is a reference to an object in
the Content Browser, not to a file on disk; the plugin works with the file system, not the
asset registry. To send an asset's contents, first get its bytes (`Export To Bytes` for a
texture, say) and pass them to the **S3 Upload Bytes** node — it accepts a byte array directly,
with no file on disk involved at all.

**When it's better to build the path with nodes rather than write it as a string.** If the
folder depends on the platform or on a save slot's name, it's more convenient to assemble an
absolute path right in the graph:

```
Get Project Saved Directory → Append ("Screenshots/") → Append (SlotName) → Append (".png")
```

**Get Project Saved Directory** (look under category `Utilities|Paths`) returns an absolute
path to `Saved/` directly, so you don't have to think about relativity beyond this. Nearby are
**Get Project Content Directory** and **Get Project Directory** — for paths inside `Content/`
and at the project root, respectively.

### S3 Upload File

Sends a file from disk.

| Pin | Description |
|---|---|
| `Bucket Name` | The bucket, e.g. `my-game-saves` |
| `Object Key` | The key, e.g. `saves/player1.sav`. No leading slash, case-sensitive |
| `Local File Path` | The file's path — absolute, or relative to the project root, e.g. `Saved/Screenshots/shot.png`. Rules and examples are in the section above |
| `Content Type` | MIME type: `image/png`/`image/jpeg` for a picture, `application/json` for a manifest, `text/plain` for text. Empty means binary data, sent as `application/octet-stream` |
| `User Metadata` | Arbitrary key-value pairs stored next to the object, e.g. `{"author": "player_42"}` |

Large files automatically go through multipart upload, several parts at once. The file is read
as it's sent, so its size never turns into memory usage.

### S3 Upload Bytes

The same, but from an in-memory array — a screenshot, a serialized save, a generated file. For
anything already on disk, use `S3 Upload File`.

> **About metadata.** S3 lowercases metadata keys. Writing `GameVersion`, you'll read back
> `gameversion`. The plugin does the same on write, so writing and reading stay symmetric.

#### Sending text: the `Data` pin expects a byte array, not a string

Unreal has no built-in "string to bytes" node — a real gap that practically everyone hits the
first time they want to put a small text file in a bucket. The plugin closes it with two pure
nodes:

```
"Welcome" ──► String To UTF-8 Bytes ──► Data  (S3 Upload Bytes)
```

**String To UTF-8 Bytes** converts an `FString` into a `TArray<uint8>` — exactly the type the
`Data` pin expects. The result for `"Welcome"` is exactly seven bytes, one per ASCII character
(`W`, `e`, `l`, `c`, `o`, `m`, `e`), with no byte-order mark (BOM). These are exactly the bytes
that end up in the object: open it later in a text editor or in your provider's console, and
you'll see the word `Welcome`, nothing more.

The full graph for uploading a text file with the word `Welcome`:

```
Event BeginPlay
   │
   ├─► Get S3 Subsystem → Get Default S3 Client ──┐
   │                                                │
   └─► "Welcome" → String To UTF-8 Bytes ───────────┼─► S3 Upload Bytes
                                                     │        Bucket Name  : my-bucket
                                                     │        Object Key   : welcome.txt
                                                     │        Data         : (from String To UTF-8 Bytes)
                                                     │        Content Type : text/plain
                                                     │
                                                     ├─ On Success → Print String "Done"
                                                     └─ On Failure → Print String (Get S3 Diagnostic Hint)
```

Set `Content Type` explicitly to `text/plain` — an empty value gets sent as
`application/octet-stream`, and while that doesn't change the file's contents, some providers
and browsers will open such an object as a download instead of showing the text on the page.

**UTF-8 Bytes To String** is the reverse node, for the opposite task: reading back a small text
object after `S3 Download Bytes`.

```
S3 Download Bytes ──► Data ──► UTF-8 Bytes To String ──► Print String
```

Both nodes work correctly with non-Latin characters too: `"Hello"` in Cyrillic converts to more
bytes than characters (each Cyrillic letter takes two bytes in UTF-8), and unfolds back into
the same string just as precisely.

---

## Downloading

### S3 Download File

Writes the object straight to a file, as it arrives. Memory doesn't depend on the size,
progress moves smoothly rather than jumping from zero to done. The `Data` pin is empty on
success — the bytes are in the file. A failed download never leaves a partially-written file
behind.

`Local File Path` here is the same absolute-or-project-relative path as in `S3 Upload File`;
rules and examples are in the [«File Path»](#file-path-absolute-or-relative) section above.
Missing folders along the path are created; an existing file at that path is overwritten.

### S3 Download Bytes

Returns the contents in memory. Good for anything you'll use right away: a texture, a config, a
small save.

### S3 Download Range

Part of an object — a file header, a preview, a single record.

| Pin | Description |
|---|---|
| `Range Start` | The first byte, from zero |
| `Range End` | The last byte, **inclusive**. `-1` means to the end of the file |

The result includes `Total Object Size` — the object's full size, even if you only read a few
bytes.

### S3 Download File Chunked

Reads the object as a series of range requests, sequentially, and writes the result to a file.

| Pin | Description |
|---|---|
| `Local File Path` | The same absolute-or-project-relative path as in `S3 Download File` |
| `Chunk Size Bytes` | The size of one request. `0` takes the part size from the settings (`Multipart Part Size Bytes`) |

This is the **only download node that can resume an interruption**: ranges let it ask for just
what's missing, while `S3 Download File` is a single streamed request that starts over. So for
a file where losing progress at the halfway point matters, use this one. The second reason to
reach for it is a provider that doesn't report the object's length up front.

Bytes land in `<file>.s3part` next to it and only move into place once fully assembled. If the
object was rewritten mid-download, the node won't stitch two versions together: `On Failure`
fires with `Precondition Failed`. Details in
[5. Transfers](05-Transfers.md#resuming-an-interrupted-download).

The `On Progress` pin here fires **once per range**, not byte by byte: the first call may
arrive with `bTotalKnown = false` still set — the plugin only learns the object's total size
from the response to the first request.

---

## Listing

### S3 List Objects

One page of a listing.

| Pin | Description |
|---|---|
| `Prefix` | Only keys starting with this. Empty means everything |
| `Delimiter` | `/` to group by "folder"; empty for a flat listing |
| `Max Keys` | Page size, from 1 to 1000 |
| `Continuation Token` | Empty for the first page |

For a "file manager" style view, pass `/` as the delimiter: `Common Prefixes` in the result will
hold the "folders," and `Objects` the files at that level.

When the result says `Is Truncated`, call the node again, passing
`Next Continuation Token` from the previous result.

### S3 List Buckets

Lists the account's buckets. Requires account-level permission, which keys scoped to "one
bucket" usually don't have, and not every provider implements it — [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) always
answers with a failure. If you already know the bucket you need, this node isn't necessary.

---

## Metadata and deletion

### S3 Get Metadata

Size, type, last-modified time, ETag and custom metadata — without reading the object itself.

It's also the cheapest way to ask "does this object exist": a `Not Found` failure means it
doesn't.

### S3 Set Metadata

Replaces an object's metadata.

S3 can't edit metadata in place, so the node copies the object onto itself on the provider's
side. Bytes never travel to you and back, but the last-modified time updates, and **any
metadata not listed in the call disappears**.

### S3 Copy Object

Copies an object entirely on the provider's side — bytes never pass through the player or
server that called the node. The same mechanism `S3 Set Metadata` uses to replace metadata
without resending content is used here directly.

| Pin | Description |
|---|---|
| `Source Bucket` / `Source Key` | Where to copy from |
| `Destination Bucket` / `Destination Key` | Where to. The destination bucket can match the source or be a different one the same keys can reach |

The destination is overwritten entirely: there's no such thing as a partial content merge.

### S3 Delete Object

Deletes a single object. Deleting something that doesn't exist counts as success: the end state
is the requested one either way.

### S3 Delete Objects (Batch)

Deletes many objects in the smallest number of requests — a thousand keys per call.

Individual keys can be rejected while the request itself succeeds. In that case
**On Success** fires with a `Partial Success` result — check `Error Count` and the `Results`
array, don't assume everything's gone.

### S3 Create Bucket

Creates a bucket. Needed on providers that don't create them on the fly — [MinIO](https://min.io/) in
particular. On [Amazon S3](https://aws.amazon.com/s3/), bucket names are unique across all
customers, so a plausible name is often already taken.

### S3 Delete Bucket

Deletes a bucket — the bucket itself, not the objects in it.

The provider refuses as long as the bucket isn't empty: `On Failure` with the error code
`BucketNotEmpty`. The node deliberately doesn't empty the bucket on its own — that's a separate,
irreversible action worth doing on purpose, not as a side effect of deleting a bucket:

```
S3 Delete Bucket
   │
   └─► On Failure (ErrorCode == "BucketNotEmpty")
          │
          ▼
       S3 Delete Objects (Batch)  ← empty it on purpose
          │
          ▼
       S3 Delete Bucket           ← try again
```

On [Wasabi](https://wasabi.com/), the account can additionally block deletion of even an empty bucket through its own
security feature ("Security Contacts") — this isn't a request error; details in
[8. Providers](08-Providers.md#wasabi).

---

## Tags

### Tags or Metadata: Which One to Use

Both are key-value pairs stored next to an object, which is exactly why they get confused. The
difference is one thing, but it decides the whole choice:

| | **User Metadata** | **Tags** |
|---|---|---|
| When they're set | While writing the object | Any time afterward |
| Changing them later | Only by rewriting the object (`S3 Set Metadata` copies it onto itself) | With one request, the object doesn't change |
| Last-modified time | Updates | Doesn't change |
| Lifecycle rules | Can't see them | Can select objects by tag |
| Access policies | Can't see them | Can grant access by tag |
| Limits | ~2 KB total for all metadata | 10 tags, key up to 128 characters, value up to 256 |

The practical rule: **what doesn't change goes into metadata, what does goes into tags.** Who
uploaded a file and with which game version — metadata. Moderation status, "flagged for
deletion," "season 2" — tags.

### S3 Get Object Tags

Reads an object's tags.

| Pin | Description |
|---|---|
| `Object Key` | The object's key, e.g. `saves/player1.sav` |
| `Result` → `Tags` | A tag map. Empty on success just means there are no tags — not an error |

### S3 Set Object Tags

Replaces an object's tags **without rewriting the object itself** and without changing its
last-modified time.

| Pin | Description |
|---|---|
| `Tags` | The full replacement set, e.g. `{"status": "verified", "season": "2"}` |

> **This replaces rather than merges.** Anything missing from the set you pass disappears. To
> add one tag to the existing ones — read the current set with `S3 Get Object Tags` first, add
> what you need to the map, and write the result back.

Values can contain any characters — ampersands, quotes, Cyrillic: the plugin escapes them in
the request's XML document and unescapes them back on read, so what you wrote is what you get
back.

### S3 Delete Object Tags

Removes all of an object's tags in one request. Removing tags from an object that had none is a
success.

### Typical usage

```
A player submits a screenshot for moderation
   │
   └─► S3 Upload File                          (the file itself + immutable metadata)
          │
          └─ On Success → S3 Set Object Tags   {"moderation": "pending"}

A moderator approves it
   │
   └─► S3 Set Object Tags   {"moderation": "approved"}
          (the object isn't rewritten, the last-modified time stays the same)
```

From there, a lifecycle rule can automatically delete everything tagged `moderation=rejected`
after thirty days — this is exactly the scenario tags exist for, separately from metadata. How
to set up a rule like that is next.

---

## Lifecycle Rules

A lifecycle rule is an instruction you give a **bucket** once, and the provider carries it out
on its own, on its own schedule. Nothing gets called from the game or the server: objects just
start living by the rules.

> **The schedule is approximate.** A provider typically sweeps a bucket about once a day, so an
> object doesn't vanish the instant it becomes old enough. Treat the timeframe as "no later
> than roughly," not as a timer.

> **There's no undo.** A rule with an expiration **deletes** objects. There is no trash can.

### S3 Set Incomplete Upload Cleanup

The most important node in this section, and the one worth running once for every bucket.

```
S3 Set Incomplete Upload Cleanup
    Bucket Name : my-game-saves
    After Days  : 7
```

**Why.** An interrupted multipart upload leaves the parts already sent in the bucket. They're
**billed as storage** and **invisible in an object listing** — meaning you can pay for them for
years without knowing.

For this plugin, this matters more than usual: an interrupted upload is deliberately **not**
canceled, so it can be resumed (see [5. Transfers](05-Transfers.md)). This exact rule is what
sweeps away the uploads nobody ever came back to finish.

Set `After Days` with some margin over the longest upload that would still be worth resuming.
Seven days is a sane default.

> This node replaces the bucket's **entire** rule set. If the bucket already has other rules
> worth keeping, use `S3 Set Bucket Lifecycle` instead.

### S3 Get Bucket Lifecycle

Reads a bucket's rules. An empty list on success means there are no rules — not an error.
(The provider actually answers a request like this with an error code; the plugin turns it
into a success with an empty list, since "no rules" is an answer, not a failure.)

### S3 Set Bucket Lifecycle

Replaces the rule set. Each rule is an `S3 Lifecycle Rule` struct:

| Field | Description |
|---|---|
| `Id` | The rule's name, unique within the bucket, e.g. `expire-temp-uploads` |
| `Enabled` | A disabled rule stays in the configuration but does nothing |
| `Prefix` | Only objects whose key starts with this, e.g. `temp/`. Empty matches **every** object in the bucket |
| `Tag Filters` | Only objects carrying all of these tags, e.g. `{"moderation": "rejected"}` |
| `Expire After Days` | Delete matching objects after this many days. `0` means never delete |
| `Abort Incomplete Uploads After Days` | Abort incomplete uploads after this many days. `0` means leave them alone |
| `Expire Orphaned Delete Markers` | Clean up "orphaned" delete markers on a versioned bucket — see below |

#### What an "orphaned delete marker" is, and when it actually happens

On an **ordinary, non-versioned bucket**, `Expire After Days` simply deletes the object
physically — and `Expire Orphaned Delete Markers` has nothing to do with this case, no need to
turn it on. This only applies to buckets with versioning enabled.

On a versioned bucket, the provider (per the S3 spec) doesn't delete an object right away: it
places a **delete marker** on top of it, and the actual versions remain behind it. If, later on,
every version behind that marker is also gone — cleaned up by another rule or by hand — the
marker is left pointing at nothing: **orphaned**. It still shows up in an object listing, as if
something were deleted there, forever, until something removes it.
`Expire Orphaned Delete Markers` is that "something."

The S3 spec forbids combining `Expire After Days` and `Expire Orphaned Delete Markers` in
**one** rule — so it's always a second, separate rule with the same `Prefix`/`Tag Filters` as
the first, but a different `Id`.

> **On [Amazon S3](https://aws.amazon.com/s3/), [MinIO](https://min.io/), [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) and [Wasabi](https://wasabi.com/) this field is optional** — it's
> only needed if you've deliberately enabled bucket versioning. **On [Backblaze B2](https://www.backblaze.com/cloud-storage) it's the
> opposite**: storage there is natively versioned, so `Expire After Days` without a matching
> `Expire Orphaned Delete Markers` rule (with the same `Prefix`) is simply rejected by the
> provider with `400 MalformedXML`. Details, with an example of the paired rules, are in
> [8. Providers → Backblaze B2](08-Providers.md#backblaze-b2).

> **This replaces rather than adds.** The set passed here becomes the bucket's entire
> configuration. To add a rule to the existing ones — read them first through
> `S3 Get Bucket Lifecycle`.
>
> An empty array is **rejected** by the node: wiping a bucket's configuration should be a
> deliberate act, not the side effect of an unfilled variable. There's a separate node below
> for that.

### S3 Delete Bucket Lifecycle

Removes all of a bucket's rules. The objects themselves aren't touched — the provider just
stops acting on them from then on. Whatever a rule already deleted can't be brought back.

### Example: cleaning up after yourself

```
(once, while setting up a bucket)
   │
   ├─► Make Array
   │     [0] Id = "sweep-uploads"        Abort Incomplete Uploads After Days = 7
   │     [1] Id = "expire-temp"          Prefix = "temp/"    Expire After Days = 1
   │     [2] Id = "expire-rejected"      Prefix = "screenshots/"
   │                                     Tag Filters = {"moderation": "rejected"}
   │                                     Expire After Days = 30
   │
   └─► S3 Set Bucket Lifecycle (Bucket Name: my-game-saves)
```

Three rules: unfinished uploads are swept away after a week, temporary files live a day,
screenshots the moderator rejected disappear after a month — and none of it needs a single line
of code your game has to run.

### What's not included

The plugin deliberately doesn't expose storage-class transitions (Glacier and the like): the
classes themselves differ between providers, and several don't have them at all, so a rule like
that would work on Amazon and silently fail to work everywhere else. If you need them, set them
in your provider's console — they don't conflict with rules set from here.

---

## Presigned URLs

Two nodes, and both are **pure**, green, with no execution pins: the URL is computed locally,
signed with the credentials on hand, with no request to the provider at all. That has a
consequence that trips people up at first — unlike every other S3 node, there's no
`On Started`/`On Success`/`On Failure` here: signing a URL has nothing that could fail over the
network, so there's nothing to wait for or catch.

| Pin | Description |
|---|---|
| `Method` | The method being signed for. The URL **doesn't work** with a different one |
| `Expires In Seconds` | Lifetime. The maximum is 604800 (7 days), a limit of Signature Version 4 itself. Values above that aren't rejected — they're silently clamped to the maximum, with a warning written to the log |

A typical mistake: signing a PUT URL and opening it in a browser. Browsers send GET, and the
provider rejects the signature.

### Which of the Two Nodes to Use

**`Make Presigned URL`** (static, category `S3|Operations`) takes `Client` as a separate pin and
returns a plain `FString`. It's a thin wrapper — under the hood it calls the same
`Generate Presigned URL` and returns just its `Url`; if `Client` is empty, it simply returns an
empty string instead of an "Accessed None" warning.

**`Generate Presigned URL`** — a node right on the client itself ("Target is S3Client," like
`S3 Upload File` and the rest), returns an `FS3PresignedUrlResult` struct with two fields:

| Field | What it is |
|---|---|
| `Url` | The same URL as `Make Presigned URL` |
| `Expires At` | The UTC moment the URL stops working — `UtcNow` plus `Expires In Seconds`, already computed for you |

If all you need is a string to hand the client, `Make Presigned URL` is simpler. If the UI has
to show "this link is valid for another N minutes" or schedule a refresh before it expires,
`Generate Presigned URL` computes that moment itself, instead of leaving you to subtract
`Expires In Seconds` by hand.

---

## Helper nodes

| Node | What it does |
|---|---|
| **Is S3 Success** | Whether everything succeeded |
| **Is S3 Partial Success** | Whether a batch operation succeeded partially |
| **Get S3 Error Message** | The provider's error text |
| **Get S3 Error Code** | The machine-readable code, e.g. `NoSuchBucket` |
| **Get S3 Diagnostic Hint** | **Exactly what to change.** The most useful one for logs while debugging |
| **Get S3 Result Summary** | Everything above, in one line |
| **Format Bytes** | `4398046` → `4.2 MB` |
| **Format Transfer Rate** | `3.4 MB/s` |

---

## The transfer handle

The object from the **On Started** pin.

| Node | What it does |
|---|---|
| **Cancel** | Stops the transfer |
| **Is Cancelled** | Whether a stop was already requested |
| **Is Finished** | Whether the transfer has ended |
| **Get Status** | Pending, Running, Succeeded, Failed, Cancelled |
| **Get Progress** | The full progress struct |
| **Get Progress Fraction** | From 0 to 1 — plug straight into a progress bar |

Holding onto the handle after completion is harmless: it just reports the final state. Dropping
it is fine too: the transfer runs to completion on its own regardless.

---

**Next:** [5. Transfers](05-Transfers.md)
