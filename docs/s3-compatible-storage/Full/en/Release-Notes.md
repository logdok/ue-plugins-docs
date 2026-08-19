*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

[← Back to contents](README.md)

# Release Notes

A short summary of each release: what's new, what's fixed, what to watch for when upgrading.
Newest release on top.

---

## 1.0

The first public release — a starting point, not something to upgrade from.

Upload, download, server-side copy, tags, lifecycle rules, bucket creation and deletion,
resuming interrupted transfers in both directions, six ways to supply credentials, profile
assets for multiple storages, and understandable errors instead of raw provider codes — all
available from both Blueprint and C++.

Verified end to end against **six real services** — [Amazon S3](https://aws.amazon.com/s3/),
[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/),
[Backblaze B2](https://www.backblaze.com/cloud-storage),
[Google Cloud Storage](https://cloud.google.com/storage), [MinIO](https://min.io/) and
[Wasabi](https://wasabi.com/) — including object keys containing spaces, ampersands and
non-Latin characters.

### Known limitations

- Storage-class transitions (Glacier and the like) are not exposed in lifecycle rules: the
  classes themselves differ between providers, and several don't have them at all. Set them in
  your provider's console — they don't conflict with rules set from the plugin.
- ACL helpers are deliberately absent: Amazon has disabled ACLs on new buckets by default since
  2023, and [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) doesn't
  support them at all. Access is granted through bucket policy or presigned URLs instead.
- Upload resume only covers files read from disk: data held in memory doesn't survive a process
  restart.
- Download resume applies to `S3 Download File Chunked`, not the plain `S3 Download File`:
  continuing from the middle requires range requests, whereas the plain download is a single
  streamed request. An object that doesn't return an ETag on a range read gets no resume
  support at all — there would be nothing to check the version against.
- [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) doesn't support
  listing buckets and answers with 403. Every other operation works.
- `S3 Delete Bucket` doesn't empty a bucket on its own — the provider refuses a non-empty bucket
  with `BucketNotEmpty`. On [Wasabi](https://wasabi.com/), the account can additionally block
  the deletion itself through a security feature, even for an empty bucket. Details in
  [8. Providers](08-Providers.md#wasabi).

### Upgrading

Nothing to upgrade from — this is the first release. Start with
[Quick Start](01-Quick-Start.md).

---

[← Back to contents](README.md)
