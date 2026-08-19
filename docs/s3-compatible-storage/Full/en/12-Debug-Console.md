*🇬🇧 English | [🇺🇦 Українська](../uk/12-Debug-Console.md)*

[← Back to contents](README.md)

# 12. Debug Console

A chapter about `S3CompatibleStorageDebug` — a separate, optional module with console commands
for testers and programmers: upload a file, inspect a bucket's contents or recent requests,
without writing a Blueprint graph or opening the editor.

---

## What this is, and why a separate module

`S3CompatibleStorageDebug` has type `DeveloperTool` — the same as the test module from
[9. Testing](09-Testing.md#the-plugins-automated-tests) — and, the same way, never ends up in a
Shipping build. The difference is what it's for: the test module verifies the plugin itself and
only runs in the editor; this one is a tool for the game, and **works in a packaged
Development build too**, not just in PIE.

Commands are `UFUNCTION(Exec)` on a `CheatManager` extension, so they follow the same rules as
any other cheat command: in multiplayer, the console may require `EnableCheats`, depending on
how `AllowCheats` is configured in your `GameModeBase`.

> **Unlike the test module, nothing here is simulated.** Every command goes through
> `Get Default S3 Client` — the exact same code and the exact same endpoint the packaged game
> uses. `S3Upload` genuinely uploads a file; `S3Delete` genuinely deletes an object.

---

## Commands

Every command works against the default bucket (`Get Default Bucket` — the one in Project
Settings, or in the current profile), so there's no need to name it every time.

| Command | Arguments | What it does |
|---|---|---|
| `S3Stats` | — | How many transfers are currently running on the default client, and each one's progress |
| `S3TestConnection` | — | Requests one object from the default bucket — the cheapest way to check that the endpoint and keys work, without a button in Project Settings |
| `S3ListObjects` | `[Prefix]` | Lists up to 50 objects |
| `S3Upload` | `ObjectKey LocalFilePath` | Uploads a local file to the bucket |
| `S3Download` | `ObjectKey LocalFilePath` | Reads an object into a local file |
| `S3Delete` | `ObjectKey` | Deletes a single object |
| `S3TailRequests` | `[Count=20]` | The most recently completed operations — see below |
| `S3ClearRequestLog` | — | Clears the log `S3TailRequests` reads |
| `S3ResumeRecords` | — | Local records of interrupted transfers — in both directions |
| `S3ClearResumeRecords` | — | Forgets every record above (locally only — see the warning below) |

Every command is asynchronous, just like the operation it calls: the result arrives in the
console as a separate line once it finishes, not immediately.

---

## S3TailRequests: the recent-operations log

The most useful command for a "something broke, but I don't know what" report. It shows
**every** operation that finished in this process — regardless of which client or which
Blueprint graph called it — newest first:

```
> S3TailRequests 3
LogS3: Display: S3TailRequests: 3 of 47 logged (newest first).
LogS3: Display:   [14:32:07] UploadFile - OK (200, 0 retries, 1.42s)
LogS3: Display:   [14:31:52] GetObjectTags - FAILED (404, 0 retries, 0.31s) - NoSuchKey The specified key does not exist.
LogS3: Display:   [14:30:18] MultipartUpload - OK (200, 2 retries, 8.77s)
```

The log is a 200-entry ring buffer that lives for as long as the process does: a new game or a
new level doesn't clear it — only `S3ClearRequestLog` does.

---

## S3ResumeRecords: what's waiting to be resumed

Shows what's currently sitting under `Saved/S3/Uploads/` and `Saved/S3/Downloads/` on this
machine — exactly what the next attempt at the same transfer will pick up, if
`Resume Interrupted Uploads` / `Resume Interrupted Downloads` is enabled:

```
> S3ResumeRecords
LogS3: Display: S3ResumeRecords: 1 upload(s) under Saved/S3/Uploads/, 1 download(s) under Saved/S3/Downloads/.
LogS3: Display:   up   my-game-saves/big/level_pack.pak (upload 2~AbCdEf..., part size 5242880, source 41943040 bytes)
LogS3: Display:   down my-game-saves/big/intro.mp4 -> /Users/you/Movies/intro.mp4 (12582912 of 41943040 bytes, etag 9b2cf1...)
```

An `up` line describes an [upload to storage](05-Transfers.md#resuming-an-interrupted-upload),
a `down` line a [download](05-Transfers.md#resuming-an-interrupted-download). Exactly how much
has been read comes from the size of the `.s3part` file, not from the record — the record
itself carries no such counter, so it has nothing to disagree with.

> **`S3ClearResumeRecords` only clears the local record.** Parts that already made it to the
> provider aren't touched — they keep being billed until they're completed, cancelled, or swept
> up by a lifecycle rule. That's what **S3 Set Incomplete Upload Cleanup** is for (chapter
> [4. Blueprint Operations](04-Blueprint-Operations.md#lifecycle-rules)), not this command.
> `.s3part` files stay on disk too — without their records, they simply stop being usable for a
> resume.

---

[← Back to contents](README.md)

**Next:** [13. Profiles in Practice](13-Profile-Scenarios.md)
