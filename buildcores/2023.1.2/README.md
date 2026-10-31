# BuildCores 2023.1.2 — REA artifacts

Reverse-engineering artifacts for `com.buildcores.buildcores_flutter`
(BuildCores: PC Builder & Parts, Flutter). Archived here to keep the analysis
workspace under its ~128 MB snapshot budget; re-fetch on demand.

## Provenance

| Item | Value |
|---|---|
| Package | `com.buildcores.buildcores_flutter` |
| Version | 2023.1.2 (versionCode 34), Aug 2023 |
| Source | APKPure XAPK (`d.apkpure.com/b/XAPK/...?versionCode=34&nc=&sv=25`) |
| XAPK SHA-1 | `f13bce9d2ce76e07ee0f554461ae7fa69ae0437f` — matches the checksum APKPure publishes |
| XAPK size | 31,629,435 B (matches CDN `full_size`) |
| Base APK MD5 | `f1a67a11436c1797548946910a379cfd` — byte-identical to the Aptoide copy, so the two sources agree |

`buildcores-2023.1.2.apk` is the App Bundle **base split** (no native code).
The Dart AOT payload lives only in `config.x86_64.apk`.

## Files

| File | Bytes | What it is |
|---|---|---|
| `buildcores-2023.1.2-apkpure.xapk` | 31,629,435 | Full XAPK: base + config.en + config.mdpi + config.zh + config.x86_64 |
| `config.x86_64.apk` | 20,095,461 | **The native split — carries libapp.so + libflutter.so** |
| `buildcores-2023.1.2.apk` | 11,351,334 | Base split (manifest, resources, flutter_assets) |
| `lib/x86_64/libapp.so` | 8,962,968 | **Dart AOT snapshot — the application logic** |
| `lib/x86_64/libflutter.so` | 11,109,376 | Flutter engine |

Verify with `sha256sum -c CHECKSUMS.txt`.

## Re-fetching the XAPK (no browser needed)

```bash
UA='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36'
REF='https://apkpure.com/buildcores-pc-part-picker/com.buildcores.buildcores_flutter/download'
curl -sSL -A "$UA" -e "$REF" -o buildcores-2023.1.2-apkpure.xapk \
  'https://d.apkpure.com/b/XAPK/com.buildcores.buildcores_flutter?versionCode=34&nc=&sv=25'
```

The `Referer` header **and** the exact `versionCode=34&nc=&sv=25` params are both
required. Without them the CDN 302s to the APKPure homepage. `?version=34` does
not work.

## Restoring into the workspace

```bash
mkdir -p ~/work/buildcores/apk/x86/lib/x86_64 ~/work/buildcores/evidence
cp buildcores-2023.1.2-apkpure.xapk buildcores-2023.1.2.apk config.x86_64.apk \
   ~/work/buildcores/apk/
cp lib/x86_64/*.so ~/work/buildcores/apk/x86/lib/x86_64/
cp -r evidence/* ~/work/buildcores/evidence/
```

Then, with REA on PATH:

```bash
rea identify-flutter-build ~/work/buildcores/apk/xapk/config.x86_64.apk
rea inspect-dart-aot       ~/work/buildcores/apk/xapk/config.x86_64.apk --abi x86_64
```

## Key findings

- **Snapshot hash** `adb4292f3ec25074ca70abcd2d5c7251`, Dart AOT kind 3
  (Dart 2.14 FullAOT layout). Engine built with
  `Android (7155654, based on r399163b1) clang version 11.0.5`.
- **129 `package:buildcores/...` Dart source paths** recovered from the
  isolate string pool — see `evidence/dart_sources.txt`.
- Backend is Firebase Realtime Database `https://buildcores.firebaseio.com`;
  full config is in `evidence/extracted-samples/strings.xml`.
  **Not probed** — requires the owner's permission.
- Monetisation: AdMob app id is declared even though the Play listing says
  "entirely ad-free"; no ad units or `AdActivity` are present.

## Caveats

- Third-party application binaries, archived for analysis only.
- The signature is an AOSP shared testkey (v2+v3, no v1) — the copy was
  re-signed by the redistributor, so provenance rests on the matching
  Aptoide/APKPure MD5 rather than the developer's key.
