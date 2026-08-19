# Demo Edition Limitations

This is the **evaluation edition** of the S3 Compatible Storage plugin. It installs
alongside the full plugin in the same project so you can try every feature before
buying — not as a cut-down alternative to it.

## What's the same as the full plugin

Every feature and mode of the plugin works in this edition exactly like the full one:
upload, download, listing, deletion, presigned URLs, multipart upload, chunked download,
tags and bucket lifecycle rules, against any supported provider. Nothing is stubbed out
or feature-limited — only *how much* you can use it is restricted.

## What's restricted

- **Request budget**: up to 50 requests actually sent to the provider per play session.
  Generating a presigned URL is free (it never sends a request), and cleanup traffic for
  an interrupted multipart upload is never blocked either, so nothing is left orphaned at
  the provider. Once the budget is used up, further operations fail with a clear reason
  until the session restarts.
- **On-screen watermark**: a small overlay reminds you this is the demo edition and shows
  the live request count. The same count is available from the `S3Stats` console command
  (see [Debug Console](12-Debug-Console.md)).
- **Shipping builds**: this plugin simply does not activate in a `Shipping` configuration —
  it is for evaluation in the Editor and Development builds only. A Shipping build compiles
  and packages normally with the plugin simply inert in it.

## Both editions installed at once

If you also have the full plugin installed in the same project, this demo edition detects
it and stands down automatically — no clients created, no watermark drawn — so it stays
out of your way. Uncheck **Stand Down When Full Plugin Present** in **Project Settings →
Plugins → S3 Compatible Storage (Demo)** to run both anyway.

## Upgrading to the full edition

Storage Profile assets created against this demo edition (`Demo S3 Storage Profile`) are a
distinct Asset Manager type from the full edition's (`S3 Storage Profile`) and do not carry
over automatically — recreate them against the full plugin's asset type after switching.
