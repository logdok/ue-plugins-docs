*🇬🇧 English | [🇺🇦 Українська](../uk/05-Transfers.md)*

[← Back to contents](README.md)

# 5. Transfers

A chapter about how the plugin handles large files, how to show progress, and how to give the
user a "Cancel" button.

---

## Progress

The `S3 Transfer Progress` struct arrives on the `On Progress` pin:

| Field | What it is |
|---|---|
| `Bytes Transferred` | How many bytes have moved |
| `Total Bytes` | How many are expected. `0` while unknown |
| `Percentage` | From 0 to 100 |
| `Current Part` / `Total Parts` | The part number, for multipart operations |
| `bTotalKnown` | Whether `Total Bytes` can be trusted |

`bTotalKnown` isn't there for show. A chunked download only learns the object's size from the
first response, so the very first progress call may arrive without it. While `bTotalKnown` is
`false`, show an indeterminate indicator, not a bar stuck at zero — otherwise the UI looks
frozen.

For a progress bar, it's more convenient to take `Get Progress Fraction` from the handle — it's
already a ready value from 0 to 1.

---

## Cancellation

```
(the player pressed "Cancel")
   │
   └─► Cancel (on the saved Transfer)
```

What happens next:

1. Whatever request is currently in flight gets aborted — the plugin doesn't wait for it to
   finish on its own.
2. A multi-step operation doesn't start its next step.
3. The **On Failure** pin fires, with a `Cancelled` result.

That last point matters: cancellation doesn't "vanish quietly." One of the final pins always
fires, so UI cleanup doesn't need to be written in two places.

> **About interrupted (not cancelled) uploads.** If the process simply disappeared — the
> application closed, the network dropped — the parts stay in the bucket on purpose, so the
> upload can be resumed. To keep whatever nobody resumed from being billed forever, set the
> bucket a **S3 Set Incomplete Upload Cleanup** rule — see
> [«Lifecycle Rules»](04-Blueprint-Operations.md#lifecycle-rules).

**Cancelling a multipart upload also cancels it on the provider's side.** Without this, parts
already sent would stay in the bucket: invisible in an object listing, but still taking up
space and billed until a lifecycle rule sweeps them away. The plugin handles this for you.

**A cancelled download never leaves a partially-written file at the destination.** A file
half-downloaded is worse than no file at all: any "is the file there?" check treats it as a
success. In `S3 Download File Chunked`, whatever arrived stays next to it in `.s3part`, so the
next attempt continues from there — see
[«Resuming an Interrupted Download»](#resuming-an-interrupted-download).

---

## Large files

### What happens automatically

Anything larger than `Multipart Part Size Bytes` becomes a multipart upload. There's nothing to
choose — the same threshold applies to both files and in-memory arrays.

The sequence: the plugin opens a multipart upload, sends parts (several at once), collects
their identifiers, and completes the upload. If some part fails for good, the whole upload gets
aborted on the provider's side — assembling an object with a hole in it isn't possible anyway.

### Memory

`S3 Upload File` doesn't read the file into memory. It describes it, and then:

- **a plain upload** — the transport streams the file straight from disk;
- **a multipart one** — each part names its own segment of that same file, by offset.

So memory usage is proportional to `Max Concurrent Parts × Multipart Part Size Bytes`, not to
the file's size. A 2 GB upload with typical settings costs roughly 20 MB, not 2 GB.

`S3 Upload Bytes` obviously holds the array in memory — it's already there.

### Provider limits

S3 allows no more than 10,000 parts. If a file is large enough that the current part size would
push past that, the plugin raises the part size on its own and logs it. Failing the upload on
the final part would be the worse outcome.

### Resuming an Interrupted Upload

When `Resume Interrupted Uploads` is on (the default), the next attempt to upload the same file
asks the provider which parts are already in place and sends only what's missing — instead of
starting from zero. It only works for `S3 Upload File` (data held in memory doesn't survive a
process restart) and only as long as the file and the part size haven't changed since the
interruption. Full details, with every edge case, are in
[FAQ: "Can an interrupted upload be resumed?"](11-FAQ.md#can-an-interrupted-upload-be-resumed).

---

## Downloading large files

### S3 Download File

Writes to a file as data arrives. Memory stays constant, progress moves byte by byte.

It's a single streamed request — and that's exactly why an interrupted download starts over:
continuing from the middle needs ranges. If a file is large enough that losing progress at the
halfway point matters, use `S3 Download File Chunked`.

### S3 Download File Chunked

Reads as a series of range requests — and can therefore **resume an interrupted download**.

`Chunk Size Bytes` = `0` means "take the part size from the settings."

### Resuming an Interrupted Download

When `Resume Interrupted Downloads` is on (the default), `S3 Download File Chunked` doesn't
write straight to the destination — it writes to a `<file>.s3part` file next to it. It only
moves into place, in one motion, once it's been read completely.

This gets you two things at once:

- **The destination path never holds a truncated file.** A file half-downloaded is worse than
  no file at all: any "is the file there?" check treats it as a success.
- **What has already arrived isn't lost.** A dropped connection, a timeout, the user
  cancelling — the `.s3part` stays put, and the next attempt to read the same object into the
  same file asks only for the bytes after it.

How much has already been read isn't recorded anywhere separately — it's the `.s3part` file's
own size. A counter could drift out of sync with reality (a write didn't land, the process
died, the counter went stale), while a file can't disagree with itself.

The chunk size can differ between attempts: a range starts at any byte, so there's no
requirement to cut the object the same way twice. This is different from uploading, where the
part size can't change.

> **Why this isn't "just append from the right place."** The object might have changed between
> attempts. Then appending would stitch the tail of a new version onto the head of an old one —
> and no later check would ever catch it: the size matches, the file reads fine, and data for
> that exact object combination never existed.

To prevent this, the plugin remembers the object's **ETag** (in `Saved/S3/Downloads/`) and
checks the version on every range, through three independent layers:

| Check | What it catches |
|---|---|
| An `If-Match` header on every request | The provider itself refuses (`412`) — before any of the wrong bytes travel over the network |
| Comparing the ETag in the response | A provider that accepted the condition but didn't actually enforce it |
| Comparing the size against `Content-Range` | A provider that doesn't return an ETag on a range at all |

What happens when the version doesn't match depends on when it's caught:

- **On the very first range of a resume** — meaning nothing has been written yet in this run:
  the `.s3part` is discarded, and the object is read from scratch. This is the ordinary "the
  file got updated since yesterday" case; you asked for the object, you get its current version
  in full.
- **In the middle of reading** — the object is being rewritten right now. The operation ends
  with `Precondition Failed`, and the incomplete file is deleted. Silently re-downloading would
  mean paying for the transfer twice without the caller ever knowing, and with no guarantee the
  object won't change again.

If the provider doesn't return an ETag on a range, no resume record is created: the download
still works, but it can't be resumed — continuing blind would mean the exact same stitching of
two versions.

This version check runs **always**, not only during a resume. An object rewritten in the middle
of an ordinary chunked download is the same stitching problem, just within a single operation.

Run `S3ResumeRecords` to see what's waiting to be resumed — see
[«Debug Console»](12-Debug-Console.md).

---

## Concurrent transfers

Several operations on the same client run in parallel and don't interfere with each other. Each
has its own handle, its own progress, and its own retry budget.

To stop everything at once:

```
Get S3 Subsystem → Cancel All Transfers
```

To find out whether anything is still running:

```
Get Default S3 Client → Get Active Transfer Count
```

Handy for an exit screen: don't close the application until the save has actually landed.

---

## Retries

Only failures worth retrying get retried:

| Retried | Not retried |
|---|---|
| Dropped connection, no response | 400 — malformed request |
| Timeout | 401, 403 — a problem with keys or policy |
| 429 — rate limiting | 404 — no such object |
| 500, 502, 503, 504 | 409 — already exists |

The delay doubles with every attempt and is picked at random from the range between zero and
the current ceiling. The randomness here isn't cosmetic: when a provider rate-limits a burst of
requests, they fail together — and without spreading the retries out, they'd come back together
too, reproducing the exact same burst.

The operation's result includes a `Retry Count` field — how many retries were spent. A non-zero
value on a successful operation is worth logging: it points to either channel quality or being
close to the provider's limit.

---

## A word about threads

Every callback arrives on the game thread. The plugin's transport layer guarantees this, not
each call site individually, so it's safe to touch actors, widgets and UObjects from any pin
with no marshaling at all.

---

**Next:** [6. C++ API](06-Cpp-API.md)
