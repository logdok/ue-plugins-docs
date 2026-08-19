*🇬🇧 English | [🇺🇦 Українська](../uk/07-Errors-And-Diagnostics.md)*

[← Back to contents](README.md)

# 7. Errors and Diagnostics

---

## Three layers of a response

Every result carries three things, and it's worth reading them in exactly this order.

| What | What it's for | Example |
|---|---|---|
| **Result Code** | Branching in code | `PermissionDenied` |
| **Error Code** | Logs and exact checks | `AccessDenied` |
| **Diagnostic Hint** | **What to change** | "The keys are valid, but the policy doesn't allow this action…" |

Providers answer with codes like `SignatureDoesNotMatch` — precise, but saying nothing about the
cause. And the cause is almost always a configuration mistake unrelated to the wording. The hint
translates one into the other.

```
On Failure
   └─► Print String (Get S3 Diagnostic Hint)
```

Or, in one line for a log: **Get S3 Result Summary**.

---

## Result Code

| Value | What happened | Where to look |
|---|---|---|
| `Success` | Everything worked | — |
| `Partial Success` | A batch operation went through, some items didn't | `Error Count`, the `Results` array |
| `Network Error` | The request never arrived: DNS, routing, connection refused, TLS | Endpoint, scheme, network |
| `Timeout` | The response didn't arrive in time | `Timeout Seconds`, the connection |
| `Authentication Error` | The keys were rejected | Secret, region, system clock |
| `Permission Denied` | The keys are valid, but the action isn't allowed | IAM policy or bucket policy |
| `Not Found` | No such bucket or object | Name, case |
| `Already Exists` | The bucket name is taken | A different name |
| `Precondition Failed` | The object changed during the transfer | Read it again — [5. Transfers](05-Transfers.md#resuming-an-interrupted-download) |
| `Invalid Request` | The request was rejected as malformed | Almost always a bug in your code |
| `Throttled` | The provider is rate-limiting | `Max Retries`, the number of concurrent operations |
| `Server Error` | A failure on the provider's side | The provider's status page |
| `Cancelled` | Cancelled by a call | — |

Note the difference between `Authentication Error` and `Permission Denied`. Both arrive as HTTP
403, but the fix is different for each: the first is a key problem, the second a policy
problem. The plugin tells them apart by the provider's error code.

`Precondition Failed` (HTTP 412) isn't a connection failure or a misconfiguration: the object in
storage was rewritten while it was being read. The plugin refuses to stitch two versions
together into one file and reports this instead of handing back a corrupted result.

---

## Common errors and what to do about them

### SignatureDoesNotMatch

The most common first-day error.

If the provider **isn't Amazon** and Path Style Addressing is off — that's almost certainly it.
Pick your provider from the settings list, and the addressing style gets set correctly.

If addressing is already correct, check in this order:

1. **The secret key** — a stray space at the start or end from copy-pasting.
2. **The region** — it's part of the signature and has to match the bucket's region.
3. **The system clock** — a signature is valid within a ±15-minute window.

The key identifier itself is definitely correct with this particular error: the provider
recognized it, otherwise it would have answered differently.

### RequestTimeTooSkewed

The machine's clock is more than 15 minutes off from the server. Turn on time sync.

### InvalidAccessKeyId

The provider doesn't know this key. If you're using **temporary** keys — make sure you're
passing the session token too: without it the key looks nonexistent, even when the key and
secret are correct.

### AccessDenied

The keys are valid, but the policy doesn't allow this specific action. This isn't a key
problem — fix the IAM policy or the bucket policy.

### NoSuchBucket

No such bucket exists in this region, for this account.

On [MinIO](https://min.io/) this often means something else: [MinIO](https://min.io/) doesn't
create buckets on first request. Create it with the **S3 Create Bucket** node or through the
[MinIO](https://min.io/) console.

### PermanentRedirect / HTTP 301

The bucket lives in a different region. Fix the Region field, and the endpoint adjusts itself
on [Amazon S3](https://aws.amazon.com/s3/).

### NoSuchKey

No such object exists. Keys are case-sensitive and **don't start with a slash**:
`saves/player.sav`, not `/saves/player.sav`.

### EntityTooSmall

A multipart upload part is smaller than 5 MB — the minimum S3 requires for every part except the
last. Raise `Multipart Part Size Bytes` to 5242880 or more. This shouldn't normally happen: the
plugin already raises the value to the minimum on its own.

### 403 on List Buckets

Listing buckets requires account-level permission, which keys scoped to "one bucket" don't
have. [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) doesn't
implement this operation at all and always answers with a failure — every other operation
works there fine.

If you already know the bucket you need, this operation isn't necessary.

---

## Logs

The plugin writes to the `LogS3` category.

```
Log LogS3 Verbose       // one line per request: method, address
Log LogS3 VeryVerbose   // plus every signed header
```

Or in `DefaultEngine.ini`:

```ini
[Core.Log]
LogS3=Verbose
```

`VeryVerbose` prints the `Authorization` header along with the credential scope — turn it on
when you're specifically debugging the signature, and off afterward.

The secret key never makes it into the logs. The credential description you'll see looks like
`AKIA... (temporary)` — enough to tell one profile from another, nothing more.

---

## Test buttons

The fastest way to check whether a configuration works — **Project Settings → Plugins →
S3 Compatible Storage**:

| Button | What it checks |
|---|---|
| **Test Connection** | Endpoint, addressing style, signature, bucket access |
| **List Objects** | That the bucket is readable, and what's in it |
| **Run Round Trip Check** | The full cycle: write, read, verify bytes, delete |

They use the exact same code as the packaged game, so a green result means a working
configuration, not just a syntactically valid one. The report appears right there, along with
hints for any errors.

---

## When nothing helps

1. Turn on `Log LogS3 VeryVerbose` and make one request.
2. Compare the address in the log against what you expect: is the host right, is the bucket
   where it should be, are special characters in the key encoded.
3. Take the `Request Id` from the result — providers can look it up in their own logs, and it's
   the single most useful thing to hand support.

---

**Next:** [8. Providers](08-Providers.md)
