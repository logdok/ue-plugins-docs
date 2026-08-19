*🇬🇧 English | [🇺🇦 Українська](../uk/02-Configuration.md)*

[← Back to contents](README.md)

# 2. Configuration

---

## Where things are stored

The plugin deliberately splits settings across three places, following one principle: **what
of this ends up in the packaged game?**

| Place | What's there | Where it goes |
|---|---|---|
| **Project Settings → S3 Compatible Storage** | Endpoint, region, addressing style, transfer parameters | `DefaultGame.ini` — into version control and into the build |
| **Project Settings → S3 Credentials (Editor Only)** | Your keys for working in the editor | `Saved/Config/.../EditorPerProjectUserSettings.ini` — on your machine only |
| **The local user store** | Keys entered by your application's end user | An encrypted file in the user's own folder, outside the project |

That's why the main settings page has no fields for keys. It's not an oversight: anything saved
there would end up in a file that ships inside the build and is trivially extracted.

---

## The Connection section

### Provider

The most important field on the page. Choosing a provider fills in everything that follows
from it:

- the **addressing style** — Amazon wants virtual-hosted (`bucket.host/key`), almost everyone
  else wants path-style (`host/bucket/key`);
- the **endpoint**, if the provider's is fixed;
- the **region**, if the provider requires a specific value.

Pick `Custom / Other` to set everything by hand — for a service that's not in the list.

### Endpoint URL

The base address, with scheme, no trailing slash.

Leave empty for [Amazon S3](https://aws.amazon.com/s3/) and [Google Cloud Storage](https://cloud.google.com/storage) — they have a well-known default endpoint.
For Amazon, the regional host is additionally computed from the Region field, so
`s3.amazonaws.com` is never a mistake here.

For everyone else the endpoint depends on your account or deployment, so it has to be given.

### Region

The region takes part in the signature, so it has to match what the provider expects even
where regions don't really exist. [MinIO](https://min.io/) doesn't check it, but the signature
still has to be consistent. [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) requires the literal `auto`.

### Path Style Addressing

How the bucket ends up in the address:

| Value | Address shape | Who it's for |
|---|---|---|
| Off | `https://bucket.host/key` | [Amazon S3](https://aws.amazon.com/s3/) |
| On | `https://host/bucket/key` | Practically everyone else |

Getting this wrong shows up as HTTP 403 with `SignatureDoesNotMatch` — a code that says nothing
about the cause. That's exactly why it's worth picking a provider from the list instead of
filling in the fields by hand.

### Default Bucket

The bucket the test buttons and the subsystem use. It's not a restriction — any operation can
still target a different bucket.

---

## The Credentials section

### Credential Source

Where the packaged game will get its keys from. Full details are in
[Credentials](03-Credentials.md); in short:

| Value | Best for |
|---|---|
| **Environment variables** | A dedicated server, a backend service, a build agent |
| **Local user store (encrypted)** | A single-user application where the bucket belongs to the user themself |
| **Anonymous** | Public buckets and presigned-URL scenarios |
| **Supplied in code** | A game client receiving short-lived keys from your backend |

### Local Store Profile Name

The name under which the local store keeps its keys. Give each application its own, so two
applications on the same machine don't read each other's keys.

---

## The Transport section

### Timeout Seconds

The timeout for **a single attempt**. A request that exhausts all its retries can take roughly
`(Max Retries + 1) × Timeout` plus the delays between attempts.

### Multipart Part Size Bytes

The threshold above which an upload becomes multipart, and at the same time the size of a
single part.

S3 requires at least 5 MB per part (except the last one), so smaller values are simply raised
to the minimum — otherwise the provider would reject the upload right at the end.

Larger parts mean fewer requests, but more repeated work if a part fails.

### Max Concurrent Parts

How many parts transfer at once.

`1` means strictly sequential. Higher values make much better use of the connection under high
latency, at the cost of holding that many buffers in memory at once. Beyond 8 the gains
typically level off while memory keeps growing.

### Use Streaming IO

Stream request and response bodies instead of holding them fully in memory.

With this on, uploading a 2 GB file costs a buffer the size of one part, not 2 GB, and downloads
report genuine byte-by-byte progress. Worth turning off only to rule this mechanism out during
diagnostics.

### Resume Interrupted Uploads

Whether to pick up an interrupted multipart upload from where it stopped, instead of starting
over. On by default.

The upload's identifier is remembered under `Saved/S3/Uploads/`, and the next attempt at the
same file asks the provider which parts are already in place and sends only what's missing.
This only applies to `S3 Upload File` — data held in memory doesn't survive a process restart —
and only as long as the file and the part size haven't changed since the interruption. Details
in [5. Transfers](05-Transfers.md#resuming-an-interrupted-upload).

### Resume Interrupted Downloads

The mirror setting for downloads: whether to continue an interrupted one instead of pulling the
object again from scratch. On by default.

Only applies to `S3 Download File Chunked` — continuing requires ranges, and the plain
`S3 Download File` is a single streamed request. Whatever has arrived so far sits next to the
destination as `<file>.s3part`, and it only replaces the destination once fully assembled.

The object is checked against its ETag on every range, so a rewritten object never gets
appended to an old tail. The chunk size can differ between attempts — unlike the part size
during an upload. Details in
[5. Transfers](05-Transfers.md#resuming-an-interrupted-download).

---

## The Retry section

Only failures worth retrying get retried: dropped connections, timeouts, rate limiting (429)
and server errors (5xx). Client errors — 403, 404 — are never retried, since a second attempt
would give the same result.

| Parameter | What it does |
|---|---|
| **Max Retries** | How many extra attempts after the first. `0` turns the mechanism off |
| **Retry Initial Delay Seconds** | Delay before the first retry |
| **Retry Max Delay Seconds** | A ceiling on the delay, so a long chain of retries doesn't stall for minutes |

The delay doubles with every attempt and is picked at random from the range between zero and
the current ceiling. This randomness matters more than it looks: when a provider rate-limits a
burst of requests, they fail together — and without spreading the retries out, they'd come back
together too, reproducing the exact same burst.

---

## Building configuration from code

The settings page isn't the only path. The same set can be assembled in Blueprint or in C++:

```cpp
FS3Config Config;
Config.Endpoint.Provider    = ES3Provider::MinIO;
Config.Endpoint.EndpointURL = TEXT("http://localhost:9000");
Config.Endpoint.Region      = TEXT("us-east-1");

Config.Transport.MaxConcurrentParts = 8;

// Sets the addressing style and clamps the transfer parameters to valid ranges.
Config.Normalize();

US3Client* Client = US3Client::CreateClient(Config);
```

In Blueprint there are the **Make S3 Config (Provider)** and **Make S3 Config (Custom)** nodes
for this, and **Validate S3 Config** returns a description of the problem, or an empty string.

---

## Multiple Storages: Profiles as Assets

The settings page describes **one** storage — the one the project works with most of the time.
If there's more than one, assembling `FS3Config` by hand in every graph means duplicating the
endpoint and region across every graph that touches them, and fixing them everywhere later.

**S3 Storage Profile** is that same configuration, saved as an asset you can simply reference.

### Creating one

**Content Browser → Miscellaneous → Data Asset → S3 Storage Profile.**

Give it a meaningful name — `SP_ArchiveStorage`, `SP_UserUploads` — and fill in the same fields
as on the settings page: provider, endpoint, region, addressing style, bucket, credential
source and transfer parameters.

At the top of the asset are the same four test buttons as in Project Settings, and they run
against this profile's own endpoint and credentials: a configuration should be verifiable
right where it's edited.

### Using one

```
Get S3 Subsystem  →  Get S3 Client For Profile  →  S3 Upload File
                         Profile: SP_ArchiveStorage
```

The asset plugs in like any other: into a node's pin, a Blueprint variable, a Data Table, an
array. Editing the asset updates everyone who references it.

The client is created on first access and reused after that, so calling the node from
different places in a graph is cheap. In the editor, editing the asset rebuilds the client
automatically — a fixed endpoint takes effect without restarting Play In Editor.

### Profiles vs. Get Named S3 Client

| | **Get Named S3 Client** | **Get S3 Client For Profile** |
|---|---|---|
| Configuration | assembled in the graph | lives in an asset |
| Changing the endpoint | edit the graph | edit the asset |
| Calling again with a different config | **silently ignored** — the client already exists under that name | no such trap: the asset is the configuration |
| Getting an already-created client elsewhere | **Find Named S3 Client** (name only, no `Config`) | the `Get S3 Client For Profile` node itself — it never asked for a config in the first place |
| When it's the right fit | the configuration is only known at runtime — it came from a backend or from the user | the configuration is known ahead of time |

If the endpoint and region are known at development time, a profile is almost always the
better choice.

### What a profile doesn't have — and why

**Fields for Access Key and Secret Key.** The asset **ships in the packaged game**, so a key
typed into it travels inside the build and is extracted from there exactly the way one is
extracted from `DefaultGame.ini` — the same reason the settings page has no such fields either.

A profile stores **where to get** credentials, not the credentials themselves:

| Field | What it's for |
|---|---|
| **Credential Source** | Same as in the settings: environment, local store, anonymous, from code |
| **Local Store Profile Name** | Give each storage its own name, so one storage's keys don't end up in another's |
| **Environment Variable Prefix** | Lets several storages share one environment: `ARCHIVE` here reads `ARCHIVE_ACCESS_KEY_ID` instead of `AWS_ACCESS_KEY_ID` |

### Keys for working in the editor

Below these fields are three more — **Editor Access Key Id**, **Editor Secret Access Key** and
**Editor Session Token**. These are this profile's own keys, and they're exactly what the test
buttons at the top of the asset use.

They're **not stored in the asset**: the fields are marked Transient, and the values land in
your own `EditorPerProjectUserSettings.ini`, under this asset's name — the same file and the
same guarantees as the shared **S3 Credentials (Editor Only)** section. They don't end up in
the build, and they don't end up in version control.

They need to be filled in here, rather than in the shared section, the moment you have more
than one storage. The shared key pair is one per project — it can only describe one storage. A
profile with no keys of its own falls back to the shared pair: a request to Amazon would go out
signed with a [MinIO](https://min.io/) key, and the provider would answer that no such key
exists. The error blames the key, even though the problem is the configuration.

To see which key actually went out, check the first line of the report:

```
---- https://s3.us-east-2.amazonaws.com, Static(editor keys for SP_Archive), region us-east-2, path-style off
```

`Static(editor keys for <profile name>)` — this profile's own keys. `Static(editor settings)` —
the shared pair. `Environment` — real environment variables.

These fields are only read when **Credential Source = Environment variables**: in the editor
they stand in for environment variables, while the packaged game never sees them at all and
reads real environment variables instead.

If the keys are only known at runtime, leave it as `Supplied in code` and set the provider on
the client yourself. After changing keys, call **Forget Profile Client** so the next access
creates the client anew — the client caches whatever it already resolved.

---

**Next:** [3. Credentials](03-Credentials.md)
