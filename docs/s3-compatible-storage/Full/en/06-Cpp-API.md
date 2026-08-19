*🇬🇧 English | [🇺🇦 Українська](../uk/06-Cpp-API.md)*

[← Back to contents](README.md)

# 6. C++ API

The same functionality as in Blueprint, with no intermediate layer.

---

## Adding the module

```csharp
// YourModule.Build.cs
PrivateDependencyModuleNames.Add("S3CompatibleStorage");
```

Headers you'll need:

```cpp
#include "S3Subsystem.h"   // get a ready-made client
#include "S3Client.h"      // operations
#include "S3Types.h"       // result and configuration structs
```

There's deliberately no single "umbrella" header: explicit includes don't add to compile time
for everyone who uses the plugin.

---

## Getting a client

```cpp
US3Client* Client = GetGameInstance()->GetSubsystem<US3Subsystem>()->GetDefaultClient();
```

The subsystem takes its configuration from the project settings and keeps the client alive.

If the configuration is assembled in code:

```cpp
FS3Config Config;
Config.Endpoint.Provider    = ES3Provider::CloudflareR2;
Config.Endpoint.EndpointURL = TEXT("https://<account>.r2.cloudflarestorage.com");
Config.Endpoint.Region      = TEXT("auto");
Config.Normalize();

US3Client* Client = US3Client::CreateClient(Config);
```

> **Lifetime.** `US3Client` is a `UObject`, so something has to reference it: a `UPROPERTY` or a
> `TStrongObjectPtr`. The client from the subsystem already takes care of this.
>
> Operations already running keep themselves alive on their own, so garbage-collecting the
> client mid-transfer breaks nothing — the transfer simply runs to completion.

---

## Uploading

```cpp
FS3TransferHandleRef Transfer = Client->UploadFile(
    TEXT("my-bucket"),
    TEXT("saves/player.sav"),
    LocalPath,
    TEXT("application/octet-stream"),
    {},                                   // metadata
    FS3OnUploadResult::CreateLambda(
        [](const FS3UploadResult& Result)
        {
            if (Result.IsSuccess())
            {
                UE_LOG(LogTemp, Log, TEXT("Uploaded, ETag %s, %lld bytes%s"),
                    *Result.ETag, Result.BytesUploaded,
                    Result.bWasMultipart ? TEXT(", multipart") : TEXT(""));
            }
            else
            {
                // ToLogString carries status, code, message and the hint.
                UE_LOG(LogTemp, Error, TEXT("%s"), *Result.ToLogString());
            }
        }),
    FS3OnProgress::CreateLambda(
        [](const FS3TransferProgress& Progress)
        {
            if (Progress.bTotalKnown)
            {
                UE_LOG(LogTemp, Verbose, TEXT("%.1f%%"), Progress.Percentage);
            }
        }));
```

From memory — `UploadBytes` with the same parameters, but an array instead of a path.

**About `LocalPath`.** Both an absolute path (`"C:/Users/You/save.png"`,
`"/Users/You/save.png"`) and one relative to the project root
(`"Saved/SaveGames/player.sav"`) are accepted — the second expands through
`FPaths::ConvertRelativePathToFull(FPaths::ProjectDir(), LocalPath)`, so the same string leads
to the same place in the editor and in a packaged game, on any platform. `DownloadFile` and
`DownloadFileChunked` accept a destination path the same way. This is exactly the mechanism
behind the `Local File Path` node in Blueprint — see
[«File Path»](04-Blueprint-Operations.md#file-path-absolute-or-relative) for a full breakdown
with examples for each platform.

---

## Downloading

```cpp
// To a file: streamed to disk, memory doesn't depend on the size.
Client->DownloadFile(TEXT("my-bucket"), TEXT("patches/1.2.pak"), LocalPath,
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            // Data is empty: the bytes are in the file.
        }));

// To memory.
Client->DownloadBytes(TEXT("my-bucket"), TEXT("config.json"),
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            if (Result.IsSuccess())
            {
                FString Json;
                FFileHelper::BufferToString(Json, Data.GetData(), Data.Num());
            }
        }));

// Part of an object.
Client->DownloadRange(TEXT("my-bucket"), TEXT("video.mp4"), 0, 1023,
    FS3OnRangeResult::CreateLambda(
        [](const FS3RangeDownloadResult& Result)
        {
            // Result.TotalObjectSize - the full size, even though we only asked for a kilobyte.
        }));

// As a series of range requests rather than one stream - and therefore the only download that
// can resume from where it broke off: whatever arrived stays in <file>.s3part.
// 0 takes the chunk size from the transport settings; it can differ between attempts.
Client->DownloadFileChunked(TEXT("my-bucket"), TEXT("patches/1.2.pak"), LocalPath,
    /*ChunkSizeBytes=*/0,
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            // ES3Result::PreconditionFailed - the object was rewritten mid-download, so the
            // plugin refused to stitch two versions together. Read it again.
        }));
```

---

## Listing and pagination

```cpp
void ListPage(US3Client* Client, const FString& Token)
{
    Client->ListObjects(TEXT("my-bucket"), TEXT("saves/"), TEXT("/"), 1000, Token,
        FS3OnListObjectsResult::CreateLambda(
            [Client](const FS3ListObjectsResult& Result)
            {
                if (!Result.IsSuccess())
                {
                    UE_LOG(LogTemp, Error, TEXT("%s"), *Result.ToLogString());
                    return;
                }

                for (const FString& Folder : Result.CommonPrefixes) { /* "folders" */ }
                for (const FS3Object& Object : Result.Objects)      { /* files */ }

                if (Result.bIsTruncated)
                {
                    ListPage(Client, Result.NextContinuationToken);
                }
            }));
}

// Lists the account's buckets. Amazon only, in its global form: R2 answers this call with 403.
Client->ListBuckets(
    FS3OnListBucketsResult::CreateLambda(
        [](const FS3ListBucketsResult& Result)
        {
            for (const FS3Bucket& Bucket : Result.Buckets) { /* Bucket.Name, Bucket.CreationDate */ }
        }));
```

---

## Metadata and deletion

```cpp
Client->GetMetadata(TEXT("my-bucket"), TEXT("save.sav"),
    FS3OnMetadataResult::CreateLambda(
        [](const FS3MetadataResult& Result)
        {
            // Result.ContentLength, ContentType, ETag, LastModified
            // Result.Metadata - custom metadata, keys lowercased
        }));

// Replacing metadata: the object is copied onto itself on the provider's side.
TMap<FString, FString> NewMetadata;
NewMetadata.Add(TEXT("game-version"), TEXT("1.4.0"));

Client->SetMetadata(TEXT("my-bucket"), TEXT("save.sav"), NewMetadata, TEXT("application/octet-stream"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));

// Batch deletion: up to a thousand keys per request.
Client->DeleteObjects(TEXT("my-bucket"), MoveTemp(Keys),
    FS3OnBatchDeleteResult::CreateLambda(
        [](const FS3BatchDeleteResult& Result)
        {
            // Can be PartialSuccess: the request went through, some keys were rejected.
            for (const FS3DeletedObject& Item : Result.Results)
            {
                if (!Item.bDeleted)
                {
                    UE_LOG(LogTemp, Warning, TEXT("%s: %s"), *Item.Key, *Item.ErrorMessage);
                }
            }
        }));
```

---

## Copying an object

Copies an object entirely on the provider's side — bytes never pass through the client. Source
and destination can be in different buckets.

```cpp
Client->CopyObject(
    TEXT("my-bucket"), TEXT("uploads/screenshot.png"),        // source
    TEXT("my-bucket"), TEXT("archive/2026/screenshot.png"),   // destination
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

`SetMetadata` above uses this same provider mechanism — copying the object onto itself — to
replace metadata without resending its contents.

---

## Object tags

Key-value pairs next to an object which, unlike metadata, can be changed at any time without
rewriting the object and without changing its last-modified time. Lifecycle rules and access
policies can select objects by tag specifically — they can't see metadata. The difference
between tags and metadata is covered in detail in
[«Tags or Metadata: Which One to Use»](04-Blueprint-Operations.md#tags-or-metadata-which-one-to-use).

```cpp
Client->GetObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    FS3OnTagsResult::CreateLambda(
        [](const FS3TagsResult& Result)
        {
            // Result.Tags - an empty map on success just means "there are simply no tags."
        }));

// This replaces rather than merges: anything missing from the passed map disappears.
TMap<FString, FString> Tags;
Tags.Add(TEXT("moderation"), TEXT("pending"));

Client->SetObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"), Tags,
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));

Client->DeleteObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

Values can contain ampersands, quotes or Cyrillic without any extra escaping on your part — the
plugin handles the request and response XML documents itself.

---

## Buckets and lifecycle rules

```cpp
// Only needed on providers that don't create a bucket on the first write to it.
Client->CreateBucket(TEXT("my-game-saves"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));

// The provider refuses as long as the bucket isn't empty (ErrorCode == "BucketNotEmpty") - the
// plugin deliberately doesn't empty the bucket itself; that's a separate, irreversible action.
Client->DeleteBucket(TEXT("my-game-saves"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

A lifecycle rule is an instruction the bucket carries out on its own, on the provider's own
schedule (typically once a day), with no call from the game or the server at all. The most
important practical use is cleaning up parts from interrupted multipart uploads, which the
plugin deliberately leaves in the bucket so they can be resumed (see
[5. Transfers](05-Transfers.md#large-files) and the
[FAQ on resuming uploads](11-FAQ.md#can-an-interrupted-upload-be-resumed)):

```cpp
FS3LifecycleRule SweepIncompleteUploads;
SweepIncompleteUploads.Id                              = TEXT("sweep-uploads");
SweepIncompleteUploads.AbortIncompleteUploadsAfterDays = 7;

Client->SetBucketLifecycle(TEXT("my-game-saves"), { SweepIncompleteUploads },
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

`SetBucketLifecycle` replaces the bucket's **entire** rule set — to add one to the existing
ones, read them first through `GetBucketLifecycle`. An empty array isn't meant to mean "change
nothing": to remove every rule on purpose, call `DeleteBucketLifecycle` separately.

```cpp
Client->GetBucketLifecycle(TEXT("my-game-saves"),
    FS3OnLifecycleResult::CreateLambda(
        [](const FS3LifecycleResult& Result)
        {
            // Result.Rules - an empty array on success means "no rules," not an error.
        }));

Client->DeleteBucketLifecycle(TEXT("my-game-saves"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

A full description of `FS3LifecycleRule`'s fields (`Prefix`, `TagFilters`, `ExpireAfterDays`) is
in [«Lifecycle Rules»](04-Blueprint-Operations.md#lifecycle-rules).

---

## Cancellation

```cpp
FS3TransferHandleRef Transfer = Client->DownloadFile(...);

// Later:
Transfer->Cancel();

// Or everything at once:
Client->CancelAllTransfers();
```

The handle also gives you `IsFinished()`, `GetState()` and `GetProgress()`. Holding onto it
after completion is harmless.

---

## Presigned URLs

```cpp
FS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
    TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    ES3HttpMethod::PUT, 900);

if (Url.IsSuccess())
{
    // Hand it to the client; it will upload the file with no keys of its own at all.
}
```

Computed locally, with no request to the provider.

---

## Credentials on a specific client

The shortest way to hand a client keys your backend just returned:

```cpp
Client->SetStaticCredentials(
    Response.AccessKeyId,
    Response.SecretAccessKey,
    Response.SessionToken,
    /*ExpiresInSeconds=*/3600);   // 0 - keys with no expiration
```

Applies to whichever client it's called on — including a profile's client from
`Get S3 Client For Profile`. That's what sets it apart from `US3Subsystem::SetRuntimeCredentials`,
which only configures the default client. Also available from Blueprint, under the same name.

The keys are kept only in memory; the plugin stops using them a little before the stated
lifetime, so a request signed right at the end doesn't arrive after it has already expired.

---

## A custom credentials provider

When keys need to refresh themselves rather than being set once — for a game client receiving
short-lived keys from your backend:

```cpp
#include "Auth/S3CredentialsProviders.h"

auto Provider = MakeShared<FS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
    FS3CredentialsProvider_Callback::FS3CredentialsFetch::CreateLambda(
        [](FS3CredentialsResolved OnFetched)
        {
            MyBackend::RequestS3Credentials(
                [OnFetched](bool bOk, const FStsResponse& Response)
                {
                    OnFetched.ExecuteIfBound(bOk, FS3Credentials(
                        Response.AccessKeyId,
                        Response.SecretAccessKey,
                        Response.SessionToken,
                        Response.ExpiresAtUtc));
                });
        }));

Client->SetCredentialsProvider(Provider);
```

A provider refreshes keys before they expire and coalesces concurrent requests: twenty
transfers on expired keys at once give your backend a single request.

Ready-made implementations: `_Static`, `_Environment`, `_LocalUserStore`, `_Callback`,
`_Anonymous`, and `FS3CredentialsProviderChain`, which tries them in order.

---

## Swapping the transport for tests

```cpp
Client->SetHttpTransport(MyFakeTransport);
```

This exact seam is what lets you test retries, multipart upload sequencing and cancellation
without a network. Details in [Testing](09-Testing.md).

---

**Next:** [7. Errors and Diagnostics](07-Errors-And-Diagnostics.md)
