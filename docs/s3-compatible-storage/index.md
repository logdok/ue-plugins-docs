# S3 Compatible Storage

<!-- last-synced:start -->
<p style="text-align: right; font-size: .75rem; opacity: .7;"><em>Docs last synced: 2026-08-17 01:04 UTC</em></p>
<!-- last-synced:end -->

Object storage for **Unreal Engine 5.7** on any S3-compatible provider — Amazon S3, Cloudflare R2, Backblaze B2, Google Cloud Storage, MinIO, Wasabi, DigitalOcean Spaces. Upload, download, list, delete and presign from Blueprint or C++, with streaming transfers, parallel multipart uploads, cancellation and automatic retries. No third-party dependencies.

Every operation is verified end to end against **five real providers**, including object keys containing spaces, ampersands and non-Latin characters.

| | Українська | Covers |
|---|---|---|
| **Full** — the complete plugin | [Посібник](Full/uk/README.md) | Everything: setup, credentials for each deployment shape, the Blueprint node reference, transfers, the C++ API, diagnostics, per-provider notes, testing |
| **Demo** — evaluation build | [Посібник](Demo/uk/README.md) | The same guide with demo-edition naming (`UDemoS3Client`, …), plus what the demo specifically limits |

!!! note "English documentation is not published yet"
    The Ukrainian guide is complete; the English translation is still being prepared. The plugin itself, its Blueprint node names, its settings and its tooltips are all in English.

## Which edition is this for?

Both editions expose an identical feature set — every chapter in the Full guide has a matching chapter in the Demo guide, just with `Demo`-prefixed type names. The only *behavioral* differences — a 50-request-per-session budget, an on-screen watermark, no talking to a provider in Shipping builds — are covered in the Demo guide's **[Обмеження демо-версії](Demo/uk/00-Demo-Limitations.md)** chapter. If you're not sure which one you're using, check your project's `Plugins/` folder: `S3CompatibleStorage` is the full edition, `S3CompatibleStorageDemo` is the demo. Both editions can be installed in the same project at once, side by side — the demo detects the full plugin automatically and stands down rather than double up.

## Where to start

Three nodes get you from a fresh install to a file in storage:

```
Get S3 Subsystem  →  Get Default S3 Client  →  S3 Upload File
```

Configure the connection once under **Project Settings → Plugins → S3 Compatible Storage**, press **Test Connection**, and the same client is available everywhere in the project.

The chapter most worth reading before writing any code is **[Облікові дані](Full/uk/03-Credentials.md)** — where credentials belong for each deployment shape, and why anything embedded in a shipped client can be extracted regardless of how it is stored.
