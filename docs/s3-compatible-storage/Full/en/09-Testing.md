*🇬🇧 English | [🇺🇦 Українська](../uk/09-Testing.md)*

[← Back to contents](README.md)

# 9. Testing

A chapter about how to test the plugin, and how to test your own integration with it.

---

## The plugin's automated tests

The plugin ships with a test suite in the `S3CompatibleStorageTests` module. The module has
type `DeveloperTool`: it builds for the editor and for Development configurations, and **never
ends up in a Shipping build**.

### Running from the editor

**Window → Test Automation**, filter `S3.`

### Running from the command line

```bash
UnrealEditor-Cmd YourProject.uproject \
  -ExecCmds="Automation RunTests S3.;Quit" \
  -unattended -nopause -nullrhi -abslog=/tmp/s3tests.log
```

Narrow the filter to run less: `Automation RunTests S3.SigV4`.

> If the project's path contains a space, pass it as a plain, shell-quoted positional argument.
> Don't add quotes inside the argument itself — the engine re-quotes arguments on its own, and
> doubled quotes make the project fail to open at all, with the run reporting "No automation
> tests matched."

### What the tests cover

94 offline tests, no network, fully deterministic:

| Group | About |
|---|---|
| `S3.Crypto.*` | SHA-256 and HMAC against published test vectors |
| `S3.SigV4.*` | Signature construction against an independent implementation of the same spec |
| `S3.UriEncode.*` | RFC 3986 encoding, including characters outside the basic plane |
| `S3.RequestBuilder.*` | Address construction, both addressing styles, encoding exactly once |
| `S3.Xml.*` | Response parsing, entities, namespaces |
| `S3.Headers.*` | ETag, Content-Range, user metadata |
| `S3.Result.*`, `S3.Diagnostics.*` | Error classification and diagnostic hints |
| `S3.Retry.*` | The retry schedule, and what does and doesn't get retried |
| `S3.Credentials.*` | Providers, the chain, key refresh, the encrypted store, when a key gets discarded after a failure and when it doesn't |
| `S3.Config.*` | Provider presets, endpoint computation, transfer-parameter sanitization |
| `S3.LocalPath.*` | Expanding a relative `Local File Path` against the project root |
| `S3.Download.*`, `S3.ChunkedDownload.*` | Downloading to a file, retries, byte-by-byte progress, assembling ranges |
| `S3.Multipart.*`, `S3.Cancel.*` | Multipart upload: every part exactly once, correct behavior when a part fails, cancellation — including a genuine DELETE abort at the provider, not just a silent stop |
| `S3.ResumeUpload.*` | Resuming an interrupted upload: skipping already-sent parts, refusing to resume a changed file |
| `S3.ResumeDownload.*` | Resuming an interrupted download: continuing from `.s3part` instead of pulling again, restarting on a changed object, refusing to stitch two versions together — including when the provider ignores `If-Match` or returns the whole object instead of a range |
| `S3.BatchDelete.*` | Splitting into batches of 1000 keys, partial failures, an empty list |
| `S3.Copy.*` | Encoding the source in `x-amz-copy-source`, a late failure in a 200-status response body |
| `S3.DeleteBucket.*` | Deletion is addressed to the bucket itself, not a key in it; a non-empty bucket is rejected with `BucketNotEmpty` and a hint about what to do next |
| `S3.Tags.*`, `S3.Lifecycle.*` | Object tags and lifecycle rules: reading, writing, "no rules" isn't an error |
| `S3.TextBytes.*` | `String To UTF-8 Bytes` / `UTF-8 Bytes To String` |
| `S3.Profile.*` | `S3 Storage Profile`: configuration construction, credential source, editor keys |
| `S3.Subsystem.*` | The subsystem's caching of named clients |
| `S3.Live.*` | An end-to-end cycle against a real service — see below |

---

## End-to-end tests against a real service

The `S3.Live.*` group is skipped and counted as passing if no environment variables are set —
so a normal run stays offline and fast.

```bash
export S3_TEST_PROVIDER=MinIO
export S3_TEST_ENDPOINT=http://localhost:9000
export S3_TEST_BUCKET=my-test-bucket
export S3_TEST_ACCESS_KEY=...
export S3_TEST_SECRET_KEY=...
export S3_TEST_REGION=us-east-1          # optional

UnrealEditor-Cmd YourProject.uproject \
  -ExecCmds="Automation RunTests S3.Live;Quit" \
  -unattended -nopause -nullrhi
```

Credentials are read **only** from environment variables: they're not in the plugin's
repository, and can't be.

Nine tests. Eight run against the same bucket (`S3_TEST_BUCKET`); `CreateAndDeleteBucket` runs
against its own, created and deleted specifically for this test, so it needs account-level
permission to create and delete buckets, not just permission on one bucket:

| Test | What it checks |
|---|---|
| `S3.Live.RoundTrip` | Write, read, metadata, a range request, listing and batch deletion — with a key containing a space, an ampersand, Cyrillic and a plus sign. A key like that is exactly what surfaces URL-encoding and XML-entity bugs that an offline test can't prove |
| `S3.Live.MultipartUploadRoundTrip` | A file larger than the part threshold genuinely goes out as several `UploadPart` calls, not one `PutObject` |
| `S3.Live.MultipartCancelAbortsAtProvider` | Cancelling mid-upload genuinely sends `AbortMultipartUpload` and leaves no parts at the provider |
| `S3.Live.CopyObjectRoundTrip` | Server-side copy |
| `S3.Live.TagsRoundTrip` | Tags: write, read, delete |
| `S3.Live.BucketLifecycleRoundTrip` | Lifecycle rules: write, read, delete |
| `S3.Live.PresignedUrlRoundTrip` | A presigned URL that genuinely works for an ordinary HTTP client with no knowledge of the plugin's signature |
| `S3.Live.ResumeInterruptedDownload` | Resuming a download against a real provider — and that a rewritten object forces a restart from scratch instead of an old tail getting a new head stitched onto it. An offline test can't prove this: it checks the code against a fake that already agrees with it, whereas `If-Match` on a ranged `GET` is something each of the six providers implements on its own |
| `S3.Live.CreateAndDeleteBucket` | Creating a bucket, deleting an empty one, refusing to delete a non-empty one with `BucketNotEmpty` and a clear hint, deleting after emptying it |

> **`S3.Live.BucketLifecycleRoundTrip` replaces the bucket's entire rule set, and deletes it
> entirely at the end** — exactly the way `Set Bucket Lifecycle` / `Delete Bucket Lifecycle`
> are supposed to behave. Don't run this test against a bucket that already has lifecycle rules
> you need.

> **`S3.Live.CreateAndDeleteBucket` genuinely creates and deletes a bucket in your account.** On
> a provider with a scoped key (a B2 application key limited to one bucket, say), the test fails
> at `S3 Create Bucket` with `403 AccessDenied` — that's about the key's permissions, not the
> plugin. On [Wasabi](https://wasabi.com/), the account can additionally block the deletion step
> itself through the "Security Contacts" security feature, even for a freshly-created empty
> bucket — see
> [8. Providers → Wasabi](08-Providers.md#wasabi). In either case, a bucket created earlier may
> be left in the account and need manual cleanup.

Not every test cleans up after itself: `S3.Live.RoundTrip` deletes everything it created, even
when a step in the middle fails, while the rest deliberately leave a few small objects under the
`ue-plugin-tests/` prefix — so the result can be inspected in the provider's console. That's
convenient for a one-off check; for a regular CI run, either sweep that prefix as a separate
step, or just accept that the bucket accumulates test debris over time.

---

## Live testing in your own project

`S3.Live.*` isn't only for developing the plugin itself. It's the fastest way to prove to
yourself that the plugin genuinely works with your own provider, account and bucket — before
you rely on it in your game. You almost certainly have a different endpoint, different keys and
a different access policy than any environment the plugin's authors verified against.

The mechanism is the same: the environment variables from the table above, pointed at **your**
service instead of [MinIO](https://min.io/). The `S3CompatibleStorageTests` module ships with
the plugin's full source, so there's nothing extra to install.

### Step by step

If you haven't done this before — it all happens in one terminal window, one step right after
the other.

**1. Open a terminal.**
macOS: `Cmd + Space`, type "Terminal". Windows: "Start" → "PowerShell".

**2. Set the environment variables** — paste and press Enter. They only apply in this window:
close it, and you'll have to enter them again.

```bash
export S3_TEST_PROVIDER=MinIO
export S3_TEST_ENDPOINT=http://localhost:9000
export S3_TEST_BUCKET=my-test-bucket
export S3_TEST_ACCESS_KEY=...
export S3_TEST_SECRET_KEY=...
export S3_TEST_REGION=us-east-1
```

Windows PowerShell syntax is different — each line becomes
`$env:S3_TEST_PROVIDER = "MinIO"`.

**3. In the same window, run the tests**, substituting your engine's real path to
`UnrealEditor-Cmd` and your project's real `.uproject` path:

```bash
"/path/to/Engine/Binaries/Mac/UnrealEditor-Cmd" \
  "/path/to/YourProject.uproject" \
  -ExecCmds="Automation RunTests S3.Live;Quit" \
  -unattended -nopause -nullrhi -abslog=/tmp/s3livetests.log
```

This takes seconds to minutes, depending on how many tests are in the `S3.Live` group and how
quickly your provider responds.

**4. Read the result.** The terminal itself scrolls past a lot of engine boilerplate, so it's
easiest to filter it:

```bash
grep -E "Test Completed|LogS3:" /tmp/s3livetests.log
```

Or open the whole file: `cat /tmp/s3livetests.log`, or by eye through Finder —
`Cmd + Shift + G`, type `/tmp` (a hidden folder by default, not reachable by a plain double
click).

> **`/tmp` isn't guaranteed to survive a machine reboot.** If you need the log around for a
> while, point `-abslog` at your home folder instead of `/tmp`, e.g.
> `-abslog=/Users/you/Desktop/s3livetests.log`.

A few practical tips:

- **Use a disposable or test bucket**, not the one holding real data. The tests write, read and
  delete objects for real — against an actual account, not a simulation.
- **The keys need permission for whatever you're checking.** A read-only key won't pass
  `S3.Live.RoundTrip` at all; a key without `PutBucketLifecycleConfiguration` won't pass
  `S3.Live.BucketLifecycleRoundTrip`, even though the rest of the tests do. This isn't a plugin
  failure — it's exactly what the check was meant to show: whether you have the permissions for
  what you're planning to do.
- **Narrow the filter** if you don't need the whole set:
  `Automation RunTests S3.Live.RoundTrip` runs just the basic cycle, without multipart upload,
  tags or lifecycle rules.
- **[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) doesn't
  implement `ListBuckets`**, and some providers don't support multipart-upload cancellation the
  expected way either — if one test fails and the rest are green, check the error message:
  `Diagnostic Hint` works there the same way it does on an ordinary operation's result.

---

## Testing your own integration

### The fastest way

**Project Settings → Plugins → S3 Compatible Storage → Run Round Trip Check.**

Write, read, byte verification and deletion — with the exact same code the packaged game will
use. If this button is green, the configuration works.

### Worth checking in your own project

| Scenario | Why |
|---|---|
| A file larger than `Multipart Part Size Bytes` | The multipart path differs from the plain one |
| Cancelling mid-transfer | Make sure the UI rolls back correctly |
| A wrong key or bucket | See what your code shows the user on failure |
| A dropped connection mid-transfer | Turn off Wi-Fi halfway through a large upload |
| A key containing spaces and non-Latin characters | If object names come from user input |

The last one is worth checking without fail if keys are built from user-entered names: encoding
bugs don't show up right away and look like "the signature doesn't match," which sends the
search in entirely the wrong direction.

### Swapping the transport in your own tests

If you're writing automated tests for your own code and don't want them hitting the network,
swap the client's transport:

```cpp
Client->SetHttpTransport(MyFakeTransport);
```

Implementing `IS3HttpTransport` — a single method — is enough. From then on, your code runs
through the plugin, and you supply the responses yourself.

---

## Logs for troubleshooting

```
Log LogS3 Verbose       // one line per request
Log LogS3 VeryVerbose   // plus every signed header
```

The secret never makes it into the logs, at any verbosity.

---

[← Back to contents](README.md)

**Next:** [10. Deployment Scenarios](10-Deployment.md)
