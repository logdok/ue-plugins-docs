*🇬🇧 English | [🇺🇦 Українська](../uk/11-FAQ.md)*

[← Back to contents](README.md)

# 11. FAQ

Short answers to what's asked most often, with a link to the chapter where each question is
covered in full. If an answer here contradicts a chapter, the chapter is right.

---

## Getting started

### How many nodes does it take to upload a file?

Three:

```
Get S3 Subsystem  →  Get Default S3 Client  →  S3 Upload File
```

No need to create a client or hold it in a variable — the subsystem does that for you.
See [1. Quick Start](01-Quick-Start.md).

### What goes into Local File Path?

Either an absolute path (`C:/Users/You/Desktop/save.png` on Windows,
`/Users/You/Desktop/save.png` on Mac), or one relative to the project root
(`Saved/Screenshots/shot.png`) — the plugin expands the second option identically in the editor
and in a packaged game. An asset path like `/Game/...` doesn't work: it's a reference to an
object in the Content Browser, not a file on disk. A full breakdown with a table of examples
for each platform is in
[«File Path: Absolute or Relative»](04-Blueprint-Operations.md#file-path-absolute-or-relative).

### How do I upload text through S3 Upload Bytes? The Data pin wants a byte array, not a string

Unreal genuinely has no built-in "string to bytes" node — use the plugin's own **String To
UTF-8 Bytes** node: it converts an `FString` into the `TArray<uint8>` you need, with no
byte-order mark, so `"Welcome"` becomes exactly seven bytes and ends up in the object as the
word `Welcome`, nothing more. The reverse node is **UTF-8 Bytes To String**, for reading a small
text object back after `S3 Download Bytes`. A full graph with an example is in
[«Sending Text»](04-Blueprint-Operations.md#sending-text-the-data-pin-expects-a-byte-array-not-a-string).

### Do I need C++?

No. Every operation is available as a node, including presigned URLs, metadata and
cancellation. C++ is only needed for a custom credentials provider or a custom transport — see
[6. C++ API](06-Cpp-API.md).

### Do I need third-party libraries or the AWS SDK?

No. The plugin only uses the engine's HTTP module and its own implementation of SHA-256 and
request signing. Nothing to install.

### Does the plugin only work with Amazon?

No. Verified against [Amazon S3](https://aws.amazon.com/s3/), [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/), [Backblaze B2](https://www.backblaze.com/cloud-storage), [Google Cloud Storage](https://cloud.google.com/storage), [MinIO](https://min.io/) and
[Wasabi](https://wasabi.com/); the provider list also includes [DigitalOcean Spaces](https://www.digitalocean.com/products/spaces) (not yet verified the same way), and
`Custom` lets you point at any compatible service by hand. See [8. Providers](08-Providers.md).

Environment variables are named `AWS_ACCESS_KEY_ID` and the like by convention — the same
convention every tool reads, not because you need Amazon.

---

## Configuration

### Test Connection is green, but it doesn't work from the game. Why?

Most likely it's the keys. In the editor, the plugin may be pulling them from sections that
**don't exist in the packaged game** (deliberately: what you typed in the editor can never
travel into the build).

Check that `Credential Source` describes a way that will actually work in production, and that
the corresponding source is configured. See
[«How the Plugin Looks Up Credentials»](03-Credentials.md#how-the-plugin-looks-up-credentials-at-runtime).

### What's the difference between Project Settings and an S3 Storage Profile asset?

Project Settings describes **one** storage — the one the project works with most of the time.
A profile asset is any other one: you can plug it into a node or a variable, and it's edited in
one place.

If there's only one storage, the settings are enough. If there's more than one — profiles.
See [2. Configuration](02-Configuration.md#multiple-storages-profiles-as-assets).

### Get Named S3 Client — what goes into Client Name?

A string **you make up yourself** — it's not an identifier from S3 or from a backend, just a
key in the subsystem's internal cache. What matters is using the same string every time you
want the same client; a different name or a different case creates a second, not-yet-configured
client. To fetch an already-created client elsewhere in the graph without assembling `Config`
again — the **Find Named S3 Client** node. More detail, with a graph example, is in
[«Configuration Known Only at Runtime»](04-Blueprint-Operations.md#configuration-known-only-at-runtime).

### I changed the keys, and the plugin is still using the old ones

The client caches whatever it already resolved. Call **Forget Profile Client** (for a profile)
or **Clear Runtime Credentials** (for keys set from code) — the next access creates the client
anew.

### What goes into Region if the provider doesn't have regions?

Any consistent value; `us-east-1` works. It can't be left empty: the region takes part in the
signature even when the provider doesn't check it. For [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) the literal `auto` is
required.

### Should Endpoint URL be left empty or not?

Empty for [Amazon S3](https://aws.amazon.com/s3/) and [Google Cloud Storage](https://cloud.google.com/storage): they have a well-known default endpoint, and for
Amazon the correct regional host is computed from the Region field. For everyone else the
endpoint has to be given — it depends on your account or deployment.

---

## Errors

### `SignatureDoesNotMatch` — where do I start?

With **Path Style Addressing**. Amazon wants it off, practically everyone else wants it on, and
this error looks exactly the same either way, saying nothing about the cause. The easiest fix —
pick your provider from the list, and the value gets set correctly.

The next suspect is the region — it's part of the signature.

### `The AWS Access Key Id you provided does not exist in our records`

Three causes, in decreasing order of likelihood:

1. **The wrong key is being used** — for example, a [MinIO](https://min.io/) key went to
   Amazon. The first line of the report names the source; see
   [3. Credentials](03-Credentials.md#how-the-plugin-looks-up-credentials-at-runtime).
2. **A temporary key with no Session Token** — without the token it looks nonexistent.
3. The key was genuinely deleted, or belongs to a different account.

### `NoSuchBucket`, even though the bucket exists

Check that the bucket is genuinely in the region you specified, and that the name has no typo.
On [MinIO](https://min.io/) buckets don't create themselves — use the **S3 Create Bucket**
node, the [MinIO](https://min.io/) console, or `mc mb`.

### Listing buckets on Cloudflare R2 returns 403

R2 doesn't implement this operation. Every other operation works normally; create and browse
buckets in the Cloudflare dashboard.

### I get 403 where I expect 404

That's how providers behave when a key doesn't even have permission to see whether an object
exists or not. This is normal: check the access policy, not whether the object is there.

### `Cannot read <path>` on a file upload

There's no file where `Local File Path` points. If the path is relative (say
`Saved/Screenshots/shot.png`), it expands against the **project root**, not the current folder
in your OS's file explorer and not the folder the game itself is running from. Check the file
at `<project root>/Saved/Screenshots/shot.png` — that's exactly where the plugin looks. Details
in
[«File Path: Absolute or Relative»](04-Blueprint-Operations.md#file-path-absolute-or-relative).

### No error, but no data either

Check `Result Code`, not just the `On Success` pin. A batch delete can end in a partial
success; an empty listing is a success with zero items, not an error.
See [7. Errors and Diagnostics](07-Errors-And-Diagnostics.md).

---

## Transfers

### What file size counts as large?

Anything past `Multipart Part Size Bytes` (5 MB by default) automatically goes through
multipart upload, several parts at once. Nothing needs to be turned on.

### How much memory does uploading a 2 GB file take?

On the order of a few buffers the size of one part, not 2 GB: the file is read as it's sent.
Downloads are written to disk the same way, as data arrives.

### How do I show progress?

The **On Progress** pin is on every transfer node. It gives you bytes sent, the total size and
a fraction — enough for a progress bar. See [5. Transfers](05-Transfers.md).

### How do I cancel a transfer?

The **On Started** pin hands you a handle; save it to a variable and call **Cancel**.
Cancelling a multipart upload cancels it on the provider's side too, so parts already sent
don't linger in the bucket — invisible in listings, and billed.

### Can an interrupted upload be resumed?

Yes, for **uploading a file from disk to storage**. The plugin remembers the multipart upload's
identifier, and the next attempt at the same file asks the provider which parts are already in
place and sends only what's missing. On by default — `Resume Interrupted Uploads` in the
Transport section.

Resuming won't happen if the file changed (size or modification time) or the part size changed:
then the file would get cut differently, and the object would end up assembled from two
different versions. In that case the upload honestly starts over.

Data in memory (`S3 Upload Bytes`) isn't resumable: after a process restart, it's simply gone.

The flip side of this behavior: an interrupted upload deliberately stays in the bucket, and its
parts are billed. Set the bucket an **S3 Set Incomplete Upload Cleanup** rule, so whatever
nobody ever resumed gets swept away on its own.

### What exactly happens if a player closes the game halfway through an upload?

As an example: a file larger than 5 MB, `S3 Upload File` reaches 30% — and at that exact moment
the player quits the game, with no graceful shutdown at all.

**Up to this point, everything's fine.** The plugin cuts the file into parts (5 MB each by
default) and sends several at once (`Max Concurrent Parts`, 4 by default). As soon as the
provider confirms the start of a multipart upload and hands back an `UploadId`, the plugin
**immediately**, not at the end, writes a reminder record to the player's disk — under
`Saved/S3/Uploads/`: which bucket, which key, and which `UploadId`. Every part the provider has
already accepted physically exists there, permanently (until it's aborted) — at 30% that's,
roughly, 3 parts out of 10.

**The player quits the game.** No graceful shutdown is needed — and that's the key point:
there's nothing left worth saving, since everything that matters is already either at the
provider (finished parts) or on disk (the reminder record). Whichever part happened to be in
flight over the network at the moment of the shutdown simply breaks off halfway: the provider
never counts it as complete (the size doesn't match), so it vanishes without a trace — not
corrupted, just as though it had never been sent.

**The player launches the game again and presses "Upload"** — the same file, bucket and key.
The exact same `S3 Upload File` node, with no separate "Resume" button anywhere. The plugin, on
its own:

1. finds the reminder record;
2. checks the file: same size and modification time as at the start? If the file was edited in
   the meantime — the plugin doesn't take the risk, discards the record and uploads everything
   from scratch;
3. if it's the same file — asks **the provider itself**, not its own record, which parts it
   actually holds under this `UploadId`;
4. the answer "parts 1, 2, 3 are here" — and the plugin sends only 4 through 10; the first
   three don't go out again;
5. progress immediately shows an honest 30%, rather than starting from zero.

Once every part is in place, the plugin tells the provider to assemble a finished object from
them, and the reminder record is deleted — it's no longer needed.

If the player **never** resumes this upload at all, those three parts still sit at the provider
and get billed, simply invisible in an ordinary object listing. That's exactly what
**S3 Set Incomplete Upload Cleanup**, from the question above, exists for.

### Can an interrupted download be resumed?

Yes — with the `S3 Download File Chunked` node. On by default: `Resume Interrupted
Downloads` in the Transport section.

The plain `S3 Download File` starts over: it's a single streamed request, and continuing from
the middle needs ranges. So for a file where losing progress at the halfway point matters, use
the chunked node.

Bytes land in `<file>.s3part` next to the destination and only move into place once fully
assembled — a truncated file never appears at the destination path. How much has already been
read is the `.s3part` file's own size; there's no separate counter.

The object is checked against its ETag on **every** range (through the `If-Match` header, and
also by comparing the ETag and size in the response). If the object changed:

- caught on the first range of a resume — the `.s3part` is discarded, and the object is read
  from scratch;
- caught in the middle of reading — the operation ends with `Precondition Failed`, and the
  incomplete file is deleted.

The plugin will never stitch the tail of a new version onto the head of an old one, under any
circumstances: this kind of error is invisible both by size and by the fact that the file reads
fine.

The chunk size can differ between attempts — a range starts at any byte. This is different from
uploading, where the part size can't change.

Details in [5. Transfers](05-Transfers.md#resuming-an-interrupted-download).

### A transfer "hangs" on a slow connection

Raise `Timeout Seconds`: it applies to **a single attempt**. A request that exhausts all its
retries can take roughly `(Max Retries + 1) × Timeout` plus the delays between attempts.

---

## Multiplayer and servers

### Does the plugin replicate anything over the network?

No. It makes HTTP requests from whichever process called it. The question "how does this work
in multiplayer" comes down to "in which process am I calling this node."
See [10. Deployment Scenarios](10-Deployment.md).

### In an eight-player session, the file gets uploaded eight times

A Blueprint event runs everywhere. If the operation should happen once — guard it with
`Has Authority` (the `Switch Has Authority` node).

### Does the plugin work on a dedicated server with no world?

Yes. The delay between retries is driven by `FTSTicker`, not a world timer, so operations run
correctly while loading a map, in a console utility, and in a commandlet.

### How do I give clients access without handing out keys?

Presigned URLs: the server signs the address and gives it to the client, and the client makes
an ordinary request against it. The client is configured as `Anonymous` for this.

---

## Security

### Can I just hardcode a key into the game?

Technically, yes — practically, consider it public. To sign a request, the key has to exist in
memory in plain form; encrypting the asset doesn't help, because the decryption key sits in the
same binary.

There's one exception: a **read-only** key for genuinely public content. Ask yourself: "what
happens if it shows up on a forum tomorrow?"

### Why is there no field for a key in the settings?

Because it would get saved into `DefaultGame.ini` — a file that travels both into version
control and into the build. The same reason the profile asset has no such fields: assets ship
inside the build.

### How reliable is the local credential store?

It protects against reading the file by eye, against a leak through a backup, and against
moving the file to another machine. It does **not** protect against someone already running
code on this machine as this same user. This is the level most desktop applications offer
without a system keychain integration; implement `IS3CredentialsStore` on top of Keychain,
DPAPI or libsecret if you need more.

### Does the secret ever end up in the logs?

Never, at any verbosity level. Logs show the key's identifier and the secret's length, never
the secret itself.

---

## Miscellaneous

### Can I work with multiple storages at once?

Yes. Set up a profile asset for each and get a client with the **Get S3 Client For Profile**
node. Clients are independent: their own endpoints, regions and keys.

### What's the difference between tags and User Metadata?

Metadata is set while writing an object, and can only be changed by rewriting the object
(`S3 Set Metadata` copies it onto itself, updating its last-modified time). Tags change with
one request at any time, without touching the object — and, unlike metadata, are visible to
lifecycle rules and access policies on the provider's side.

A simple rule: **what doesn't change goes into metadata, what does goes into tags.** A
comparison table and an example are in
[«Tags or Metadata»](04-Blueprint-Operations.md#tags-or-metadata-which-one-to-use).

### I'm worried about incomplete uploads — are they really billed?

Yes. Parts from an interrupted multipart upload stay in the bucket, are **billed as storage**,
and are **invisible in an object listing** — meaning it's easy not to notice them for years. On
top of that, the plugin deliberately doesn't cancel an interrupted upload, so it can be resumed.

Run the **S3 Set Incomplete Upload Cleanup** node once for every bucket (seven days is a
sensible value): the provider sweeps away whatever nobody ever resumed. Details in
[«Lifecycle Rules»](04-Blueprint-Operations.md#lifecycle-rules).

### Can I create a bucket from the game?

Yes, with the **S3 Create Bucket** node. On [MinIO](https://min.io/) this is often necessary,
since buckets don't create themselves there. On Amazon, bucket names are globally unique across
every customer, so a plausible name is often already taken — that's normal.

### Do object keys containing spaces and Cyrillic characters work?

Yes, and this is verified end to end on all six services with exactly this kind of key — a
space, an ampersand, Cyrillic and a plus sign. If object names come from user input, test this
case in your own project too.

### How do I see the real HTTP traffic?

```
Log LogS3 Verbose       // one line per request
Log LogS3 VeryVerbose   // plus every signed header
```

### Does the test module ship in a packaged game?

No. It has type `DeveloperTool`: it builds for the editor and Development configurations, and
never ends up in Shipping.

---

**Back:** [Contents](README.md)

**Next:** [12. Debug Console](12-Debug-Console.md)
