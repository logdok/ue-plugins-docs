*🇬🇧 English | [🇺🇦 Українська](../uk/01-Quick-Start.md)*

[← Back to contents](README.md)

# 1. Quick Start

The goal of this chapter is to upload a file to storage and read it back within ten minutes,
without writing a single line of C++.

---

## Installation

1. Copy the `S3CompatibleStorage/` folder into your project's `Plugins/` directory.
2. Regenerate your project files and build.
3. Make sure the plugin is enabled: **Edit → Plugins → Networking → S3 Compatible Storage**.

---

## Step 1. Connection

**Project Settings → Plugins → S3 Compatible Storage**

Start with the **Provider** field. It's the most important one: picking a provider immediately
gives you the correct addressing style, and for some services a ready-made endpoint too. This
is usually where the reason for "nothing works even though the keys are right" is hiding.

| Provider | Endpoint URL | Region |
|---|---|---|
| [Amazon S3](https://aws.amazon.com/s3/) | leave empty | your bucket's region |
| [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) | `https://<account-id>.r2.cloudflarestorage.com` | `auto` |
| [Backblaze B2](https://www.backblaze.com/cloud-storage) | `https://s3.<region>.backblazeb2.com` | from the B2 console |
| [Google Cloud Storage](https://cloud.google.com/storage) | leave empty | `auto` |
| [MinIO](https://min.io/) | `http://host:9000` | any value, but it takes part in the signature |
| [Wasabi](https://wasabi.com/), [DigitalOcean Spaces](https://www.digitalocean.com/products/spaces) | depends on the region | depends on the region |

Fill in **Default Bucket** — it's what the test buttons and the subsystem use.

> **[Amazon S3](https://aws.amazon.com/s3/): don't type the regional endpoint by hand.** Leave
> Endpoint URL empty and just set the region. The plugin resolves `s3.<region>.amazonaws.com`
> on its own, because `s3.amazonaws.com` only serves `us-east-1`.

---

## Step 2. Keys

Scroll to the **S3 Credentials (Editor Only)** section and enter your Access Key ID and Secret
Access Key.

This section is deliberately separate from the rest of the settings. It's stored in your
personal `EditorPerProjectUserSettings.ini`, which lives under `Saved/`, is excluded from
version control by the standard `.gitignore`, and **never ends up in a packaged game**. The
main settings page has no fields for keys at all — and that's deliberate too: anything saved
there would travel inside the build, in `DefaultGame.ini`.

For a packaged game, keys come from a completely different place — see
[Credentials](03-Credentials.md).

---

## Step 3. Verification

Four buttons sit at the top of the settings page:

| Button | What it does |
|---|---|
| **Test Connection** | One real request: is the endpoint reachable, does the signature match, is the bucket readable |
| **List Objects** | Shows the bucket's contents |
| **Run Round Trip Check** | Uploads a test object, reads it back, verifies the bytes, and deletes it. Requires write access |
| **Clear Report** | Clears the report |

The result appears in the **Report** field right below the buttons. If a check fails, the
report includes not just the provider's error text but also a line suggesting what to change.

The buttons use the exact same code as the packaged game, so a green result here means the
configuration actually works — not just that it's syntactically valid.

---

## Step 4. Uploading a file from Blueprint

```
Event BeginPlay
   │
   ├─► Get S3 Subsystem
   │        │
   │        └─► Get Default S3 Client ──┐
   │                                     │
   └─────────────────────────────────────┴─► S3 Upload File
                                                  Bucket Name : my-bucket
                                                  Object Key  : saves/player.sav
                                                  Local File Path : Saved/SaveGames/player.sav
                                                  │
                                                  ├─ On Started  → save the Transfer to a variable
                                                  ├─ On Progress → update a ProgressBar
                                                  ├─ On Success  → Print String "Done"
                                                  └─ On Failure  → Print String (Get S3 Diagnostic Hint)
```

**Get S3 Subsystem** is a standard Unreal node: `Get Game Instance Subsystem` with the class
set to `S3 Subsystem`. The subsystem creates a client from your settings, keeps it alive, and
returns the same instance on every call.

**About `Local File Path`.** The value above is a relative path: the plugin expands it against
the project root on its own (`Saved/SaveGames/player.sav` becomes, for example,
`C:/MyProject/Saved/SaveGames/player.sav`), the same way in the editor and in a packaged game.
A full absolute path such as `C:/Users/You/Desktop/player.sav` works just as well. What doesn't
work is an asset path like `/Game/...` — that's a reference to an object in the Content
Browser, not a file on disk. A full breakdown of every case, with an example for each platform,
is in [«File Path: Absolute or Relative»](04-Blueprint-Operations.md#file-path-absolute-or-relative).

### What to do with the pins

- **On Started** fires immediately, before any data has moved. It gives you a `Transfer`
  object — save it to a variable if you want to give the player a "Cancel" button or draw
  progress.
- **On Progress** fires as bytes move. The struct has `Percentage` for a progress bar and
  `bTotalKnown` — while it's `false`, the overall size isn't known yet, so show an indeterminate
  indicator.
- **On Success** and **On Failure** — exactly one of the two, always. So cleanup can be written
  in one place.

All pins fire on the game thread, so it's safe to touch actors and widgets from any of them.

---

## Step 5. Reading it back

```
S3 Download File
   Bucket Name     : my-bucket
   Object Key      : saves/player.sav
   Local File Path : Saved/SaveGames/player-downloaded.sav
   │
   ├─ On Success → the file is already on disk
   └─ On Failure → show Get S3 Diagnostic Hint
```

The file is written to disk as data arrives, so memory usage doesn't depend on its size. The
`Data` pin is empty on success — the bytes are in the file. If the download fails, the
partially-written file is deleted: a half-file that looks like a successful download is worse
than no file at all.

To get the data in memory instead — for a small JSON file, say — use **S3 Download Bytes**.

---

## Common first-run obstacles

| Symptom | Cause |
|---|---|
| `SignatureDoesNotMatch` on a non-Amazon service | Path Style Addressing is disabled. Pick your provider from the list, and it turns on by itself |
| `NoSuchBucket` on [MinIO](https://min.io/) | [MinIO](https://min.io/) doesn't create buckets automatically. Use the **S3 Create Bucket** node or the [MinIO](https://min.io/) console |
| The node does nothing, no pins fire | The Client pin is empty. Take the client from the subsystem |
| 403 on everything in the packaged game | The build has no keys — they were left in the editor-only settings. See [Credentials](03-Credentials.md) |
| `Cannot read <path>` on `S3 Upload File` | There's no file at the path you gave it. For a relative path, that's `Saved/...` under the **project** root, not wherever your OS file explorer happens to be open — see [«File Path»](04-Blueprint-Operations.md#file-path-absolute-or-relative) |

---

**Next:** [2. Configuration](02-Configuration.md)
