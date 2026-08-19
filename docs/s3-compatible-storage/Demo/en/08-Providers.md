*🇬🇧 English | [🇺🇦 Українська](../uk/08-Providers.md)*

[← Back to contents](README.md)

# 8. Providers

Every service listed here implements the same S3 API, but the details differ. Picking a
provider in the settings closes most of these gaps automatically; this chapter covers what's
worth knowing beyond that.

Every provider in this chapter except [DigitalOcean Spaces](https://www.digitalocean.com/products/spaces) has been verified end to end: write, read,
metadata, a range request, listing and batch deletion — including keys containing spaces,
ampersands and Cyrillic characters. A separate end-to-end test
(`S3.Live.ResumeInterruptedDownload`) verifies resuming an interrupted download too — that
`If-Match` on a ranged `GET` genuinely makes the provider reject a stale version of an object,
rather than just ignoring the condition. This doesn't follow from the other tests: each
provider implements conditional requests on its own. One more separate test
(`S3.Live.CreateAndDeleteBucket`) verifies bucket creation and deletion, in particular that
deleting a non-empty bucket is rejected with a clear reason rather than silently emptying it.

## What works where

| Provider | Write and read | Multipart + cancellation | Copy | Presigned URLs | Tags | Lifecycle rules | Download resume | Create / delete bucket |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| [Amazon S3](#amazon-s3) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| [Cloudflare R2](#cloudflare-r2) | ✅ | ✅ | ✅ | ✅ | ❌ 501, not implemented | ✅ | ✅ | ✅ |
| [Backblaze B2](#backblaze-b2) | ✅ | ✅ | ✅ | ✅ | ⚠️ a B2 bug | ✅ | ✅ | ✅ |
| [Google Cloud Storage](#google-cloud-storage) | ✅ | ✅ | ✅ | ✅ | ❌ 400, not implemented | ❌ its own schema | ✅ | ✅ |
| [MinIO](#minio) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| [Wasabi](#wasabi) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ deletion may be blocked by the account |

✅ verified end to end against the real service · ⚠️ mostly works, with a known quirk · ❌ the
provider doesn't implement this part of the S3 API. Details are in each provider's own section
below; click a name in the table.

---

## Amazon S3

| | |
|---|---|
| Endpoint URL | leave empty |
| Region | your bucket's region |
| Path Style Addressing | off |

**Regional endpoints.** `s3.amazonaws.com` only serves `us-east-1`. Any other region has to go
to `s3.<region>.amazonaws.com`, or the provider answers with 301. The plugin substitutes the
correct host on its own — so the endpoint field can stay empty as long as you set the region.

**Bucket creation** outside `us-east-1` requires naming the region in the request body. The
plugin adds it where needed, and doesn't add it where doing so would itself be an error.

**Bucket names are globally unique** across every Amazon customer. A plausible name is usually
already taken — that's normal, not a sign of a problem.

---

## Cloudflare R2

| | |
|---|---|
| Endpoint URL | `https://<account-id>.r2.cloudflarestorage.com` |
| Region | `auto` — that exact literal |
| Path Style Addressing | on |

**Listing buckets doesn't work.** R2 doesn't implement this operation and always answers with
403. Create and browse buckets in the Cloudflare dashboard instead.

**Object tags don't work.** R2 answers `Set Object Tags` with `501 NotImplemented` — not a
permissions error, but a direct statement from the provider that this part of the S3 API is
absent. If you need tags specifically on R2, use custom metadata (`Set Metadata`) instead — not
the same mechanism (metadata is immutable without rewriting the object), but it's what R2
supports.

Everything else — upload, download, server-side copy, multipart upload and its cancellation,
presigned URLs — has been verified and works normally.

**Lifecycle rules require a separate token permission.** `Get`/`Set`/`Delete Bucket
Lifecycle` is a bucket-level action, not an object-level one, and an ordinary "Object Read &
Write" R2 token isn't enough for it: without the right permission, `Set Bucket Lifecycle`
answers with `403 Access Denied`, the same way any other action without the right permission
would. The API itself supports this fine — verified end to end, and it started working
immediately once the token's permissions were changed.

The token for this isn't created on the general **Account → API Tokens** page (there's no R2
template there at all), but specifically on the **R2 → Manage R2 API Tokens** page — a separate
one, with R2's own templates. Pick **Admin Read & Write** there (not just "Object Read &
Write") and, if you want, narrow the **Bucket scope** to one specific bucket.

**Compression of text content types.** R2 compresses responses for text `Content-Type`s on the
fly. The visible consequence: for an object like that, the response carries a *weak* ETag
(`W/"..."`) and no length at all.

The plugin accounts for this: metadata requests ask for an uncompressed representation, and
ETags get normalized, so an identifier from R2 compares correctly against the same object's
identifier on any other service. Nothing extra to do here — but if you talk to R2 outside the
plugin, it's worth keeping in mind.

---

## Backblaze B2

| | |
|---|---|
| Endpoint URL | `https://s3.<region>.backblazeb2.com` |
| Region | from the Endpoint field in the B2 console |
| Path Style Addressing | on |

The endpoint and region depend on where the bucket was created. Both are visible in the B2
console, under the bucket's properties, in the Endpoint field.

B2 calls the secret key an "application key."

**A lifecycle rule with only `Expire After Days` is rejected by B2.** A request that Amazon,
[MinIO](https://min.io/), R2 or [Wasabi](https://wasabi.com/) accept without question comes back here as `400 MalformedXML`, with
an explanation that there's an Expiration rule but no matching ExpiredObjectDeleteMarker rule
with the same prefix — B2 is natively versioned, so `Expire After Days` there always leaves
behind a delete marker instead of erasing the object outright. The full mechanism is explained
in
[4. Blueprint Operations](04-Blueprint-Operations.md#what-an-orphaned-delete-marker-is-and-when-it-actually-happens).

A working pair of rules for B2 — the same `Prefix`, different `Id`s:

```
Make Array
  [0] Id = "expire-temp"           Prefix = "logs/"    Expire After Days = 30
  [1] Id = "expire-temp-markers"   Prefix = "logs/"    Expire Orphaned Delete Markers = true
```

`Expire Orphaned Delete Markers` was added to `FDemoS3LifecycleRule` for exactly this case —
verified end to end against a real B2 bucket.

**Reading tags after deleting them on B2 can return `405`, rather than an empty list.**
`Set Object Tags` and the first `Get Object Tags` after it work normally; after
`Delete Object Tags`, that same `Get Object Tags` on the same object sometimes answers with
`405 MethodNotAllowed` instead of the expected empty tag set. This is B2's own behavior,
reproduced by the end-to-end test; if something suddenly fails with 405 on B2 right after
deleting tags where it used to work, this is why — not a regression in your own code.

**`S3 Create Bucket` / `S3 Delete Bucket` require a key with account-level permissions.**
B2's own officially documented scenario — `aws s3api create-bucket` against a B2 endpoint —
works exactly the way it does against any other S3-compatible provider. But an application key
scoped to one specific bucket (the usual practice for B2) has no permission to create a new
bucket in the account — the provider answers `403 AccessDenied`, and the plugin surfaces that
directly rather than failing silently. Create a key without a bucket restriction if you need
bucket management from code.

---

## Google Cloud Storage

| | |
|---|---|
| Endpoint URL | leave empty |
| Region | `auto` |
| Path Style Addressing | on |

The plugin talks to Google's **S3-compatible XML API**, not its native API. That means it needs
**HMAC keys**, not a service account's JSON key — in the console that's
Cloud Storage → Settings → Interoperability → Create a key.

Google rewrites the `Accept-Encoding` header, so the plugin deliberately leaves it out of the
signature. This is an implementation detail, but it's exactly why some other clients get 403
from GCS in cases where every other service works fine.

**Lifecycle rules don't work through the S3 API at all.** Even through the "S3-compatible" XML
API, GCS's `?lifecycle` endpoint expects a separate, non-S3 request body format: instead of
the standard `<Filter>`/`<Status>`/`<Expiration>`, it has its own vocabulary of
`<Rule><Action>...</Action><Condition>...</Condition></Rule>`. `Set Bucket Lifecycle` sends a
correct document per the S3 spec — the same one Amazon, [MinIO](https://min.io/) and (partly)
other providers accept — and GCS answers `400 MalformedLifecycleConfiguration`, because it's not
the document it's expecting. This isn't a plugin bug, and it isn't something a different request
encoding would fix: to manage a bucket's lifecycle on GCS, set the rules in the
[Google Cloud Storage](https://cloud.google.com/storage) console instead.

**`S3 Create Bucket` / `S3 Delete Bucket` require the `Storage Admin` role on the project.**
The role the GCS console offers by default when creating an HMAC key for interoperable
access — `Storage Object Admin` — acts at the bucket level and doesn't include the
`storage.buckets.create` permission, which is
[officially documented](https://cloud.google.com/storage/docs/xml-api/put-bucket-create) as
required for creating a bucket through the XML API; with it, the request answers
`400 InvalidArgument` (reproduced both by the plugin and independently, with the same key
through `mc mb`, with no plugin code in the path at all). Grant the key the `Storage Admin` role
at the project level instead of `Storage Object Admin` if you need bucket management from code.

Creating a bucket through the XML API additionally requires either an `x-goog-project-id`
header (the plugin doesn't send it — it's a non-standard, non-S3 field) or a default project for
interoperable access set in the **Interoperability** console — a one-time setup, after which an
ordinary request works with no extra header.

---

## MinIO

| | |
|---|---|
| Endpoint URL | `http://host:9000` |
| Region | any value, but it takes part in the signature |
| Path Style Addressing | on |

[MinIO](https://min.io/) runs anywhere — on a workstation, in your own data center, in a managed
cluster. The quirks are the same in every case.

**Buckets need to be created ahead of time.** [MinIO](https://min.io/) doesn't create a bucket
on the first write attempt: you'll get `NoSuchBucket`. Create it with the **S3 Create Bucket**
node, through the [MinIO](https://min.io/) console, or with `mc mb` — the plugin doesn't
restrict the method in any way.

**Region** isn't checked by [MinIO](https://min.io/), but the signature still has to be
consistent, so the field can't be left empty. The default value works fine.

**The scheme is `http://`**, unless you've set up TLS. This is the most common reason a local
[MinIO](https://min.io/) instance "doesn't respond."

---

## Wasabi

| | |
|---|---|
| Endpoint URL | `https://s3.<region>.wasabisys.com`, e.g. `https://s3.eu-central-1.wasabisys.com` |
| Region | the same one as in the endpoint — [Wasabi](https://wasabi.com/) doesn't redirect from a wrong regional host the way Amazon does |
| Path Style Addressing | on |

The closest thing to plain S3, with no exceptions: write, read, copy, tags, lifecycle rules and
resuming an interrupted download all passed the end-to-end test without a single caveat, where
other providers trip up one way or another.

**But the account can block bucket deletion, even for an empty bucket.** In the end-to-end
test, `S3 Create Bucket` worked normally, and the following `S3 Delete Bucket` on that same,
freshly-created and empty bucket answered `424 Failed Dependency`, explaining "Bucket Delete
Activity ... was blocked by Security Contacts in place." This is an account-level security
feature on [Wasabi](https://wasabi.com/) (protection against accidental or malicious bucket
deletion, confirmed by security contacts), not a request error — the exact same request worked
without issue on [MinIO](https://min.io/), R2 and [Amazon S3](https://aws.amazon.com/s3/). If
you need programmatic bucket deletion on [Wasabi](https://wasabi.com/), this feature has to be
turned off or confirmed in the [Wasabi](https://wasabi.com/) console beforehand; otherwise the
plugin correctly reports the refusal rather than trying to work around it.

---

## DigitalOcean Spaces

| | |
|---|---|
| Endpoint URL | `https://<region>.digitaloceanspaces.com`, e.g. `https://nyc3.digitaloceanspaces.com` |
| Region | the same one as in the endpoint |
| Path Style Addressing | on |

**⚠️ Not verified live.** Unlike every other provider in this chapter, DigitalOcean Spaces
hasn't yet passed an end-to-end test against a real account — everything below is taken from
the provider's own official documentation, not confirmed by a request from the plugin. And the
gap between what's documented and what actually happens is more the rule than the exception
here: of the six providers verified live, three
([Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/),
[Backblaze B2](https://www.backblaze.com/cloud-storage),
[Google Cloud Storage](https://cloud.google.com/storage)) turned out to diverge from their own
documentation exactly on tags and lifecycle rules — precisely what's claimed below. Treat this
as claimed, not proven.

Per the official [Spaces API reference](https://docs.digitalocean.com/reference/api/spaces/):

- **Object tags** (`Get`/`Put`/`Delete Object Tags`) are documented as supported.
- **Lifecycle rules** (`Get`/`Set`/`Delete Bucket Lifecycle`) are also documented as
  supported — but with a separate
  [documented](https://docs.digitalocean.com/products/spaces/how-to/configure-lifecycle-rules/)
  condition: a rule can't be filtered by tags, only by prefix and duration. That is, `Prefix`
  and `ExpireAfterDays` in `FDemoS3LifecycleRule` should work, but `TagFilters` likely won't.

---

## Another service

Pick `Custom / Other` and fill in the fields by hand. Almost always you'll need:

- **Path Style Addressing — on.** Virtual-hosted addressing among S3-compatible services is
  used almost exclusively by Amazon.
- **Region** — anything consistent; if the service has no regions, `us-east-1` works.

After that — the **Test Connection** button: if the configuration is wrong, the report names
the specific setting.

---

**Next:** [9. Testing](09-Testing.md)
