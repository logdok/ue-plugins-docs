# S3 Compatible Storage

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

Object storage for Unreal Engine — on any S3-compatible provider.

> **Version 1.0** · Runtime · UE 5.7 · [Release Notes](Release-Notes.md)
> Win64 · Mac · Linux · iOS · Android
> No third-party libraries: just the engine's HTTP module and our own SHA-256 implementation

Verified end to end against **[Amazon S3](https://aws.amazon.com/s3/), [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/), [Backblaze B2](https://www.backblaze.com/cloud-storage),
[Google Cloud Storage](https://cloud.google.com/storage), [MinIO](https://min.io/) and [Wasabi](https://wasabi.com/)**.

---

## Quick start: three nodes

The shortest path from "just installed" to "a file in storage."

**1. Configure the connection once**

Open **Project Settings → Plugins → S3 Compatible Storage**:

| Field | What to enter |
|---|---|
| **Provider** | Your provider from the list. This is the key field — it fills in the rest for you |
| **Endpoint URL** | Leave empty for [Amazon S3](https://aws.amazon.com/s3/) and [Google Cloud Storage](https://cloud.google.com/storage). For everyone else — your service's address |
| **Region** | Your bucket's region. For [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) — the literal `auto` |
| **Default Bucket** | The bucket you'll work with by default |

Below that, in the **S3 Credentials (Editor Only)** section, enter your keys. They're stored in
your own `EditorPerProjectUserSettings.ini` and **never end up in a packaged game** — details in
[Credentials](03-Credentials.md).

**2. Press Test Connection**

The button at the top of the page. It sends a real request using the exact same code the packaged
game will use, and prints the result right below the buttons. If something's wrong, it tells you
which setting to change.

**3. Run an operation**

```
Get S3 Subsystem  →  Get Default S3 Client  →  S3 Upload File
```

No need to create a client or store it in a variable — the subsystem does that for you.

---

## Chapters

| Chapter | About |
|---|---|
| [Release Notes](Release-Notes.md) | What's new and what's fixed in each release |
| [1. Quick Start](01-Quick-Start.md) | Your first upload and first download, step by step |
| [2. Configuration](02-Configuration.md) | The settings page, providers, transfer parameters |
| [3. Credentials](03-Credentials.md) | Where to keep your keys — and why it's the most important decision |
| [4. Blueprint Operations](04-Blueprint-Operations.md) | Every node, with pin descriptions and examples |
| [5. Transfers](05-Transfers.md) | Progress, cancellation, large files, multipart upload |
| [6. C++ API](06-Cpp-API.md) | The same functionality, from code |
| [7. Errors and Diagnostics](07-Errors-And-Diagnostics.md) | How to read errors and what to do about the common ones |
| [8. Providers](08-Providers.md) | Quirks of each service |
| [9. Testing](09-Testing.md) | The plugin's automated tests, and how to test your own integration |
| [10. Deployment Scenarios](10-Deployment.md) | Single-player, multiplayer, dedicated server, desktop app |
| [11. FAQ](11-FAQ.md) | Short answers to the questions asked most often |
| [12. Debug Console](12-Debug-Console.md) | Console commands for testers and programmers: upload, list, watch recent requests |
| [13. Profiles in Practice](13-Profile-Scenarios.md) | Profile assets in single-player, a listen server, a dedicated server and a desktop app — where each one's keys come from |
| [14. Credentials Cookbook](14-Credentials-Cookbook.md) | Step by step for every configuration, C++ and Blueprint side by side: from the key-entry screen to presigned links |

---

## What the plugin does

**Operations.** Uploading and downloading objects, partial byte-range reads, listing objects
and buckets, single and batch deletion, reading and replacing metadata, server-side copying,
bucket creation, presigned URL generation.

**Large files.** Anything larger than the part size automatically goes through multipart
upload, several parts at once. The file is read as it's sent, so a 2 GB upload costs a few
buffers, not 2 GB of memory. Downloads are written to disk the same way, as data arrives.

**Reliability.** Dropped connections, timeouts, provider-side rate limiting and server failures
are retried automatically with exponential backoff. Client errors are not retried — retrying
wouldn't make them go away.

**Cancellation.** Every operation returns a handle. Cancelling a multipart upload also cancels
it on the provider's side, so parts already sent don't linger in the bucket — invisible in
listings, and billed anyway.

**Understandable errors.** A provider answers with a code like `SignatureDoesNotMatch`, which
says nothing about the cause. The plugin adds a sentence to the result naming exactly which
setting to change.

---

## What's not included

- Storage-class transitions in lifecycle rules — set those in your provider's console.
- ACL helpers. Amazon disables ACLs on new buckets by default, and [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) doesn't
  support them at all — access is granted through bucket policy or presigned URLs instead.

---

## License

MIT.
