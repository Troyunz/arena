# BuildCores — REA findings (interim)

**Target:** BuildCores — PC Part Picker
**Package:** `com.buildcores.buildcores_flutter`
**Version:** 2023.1.2 (versionCode 34)
**Artifact:** `buildcores-2023.1.2.apk`, 11,351,334 bytes
**SHA-256:** `b271c4ad948c01e710d21d5055ba5fb5737cf066abdda0ee40c018806ef5f0a9`
**Source:** Aptoide pool (MD5 `f1a67a11436c1797548946910a379cfd` verified)
**Date:** 30 October 2026

**Tooling:** REA (`rea-agents`) 6.8.0 · Node v24.22.2 · OpenJDK 21.0.13 ·
jadx-headless-mcp 0.7.1 (SHA-256 `6e5eacf5…ff75`, verified) · apktool 2.10.0

> **Status: interim.** The Dart application payload is not in this artifact —
> see [The gap](#the-gap).

---

## Evidence tags

| Tag | Meaning |
| --- | --- |
| `[observed]` | Read directly from the artifact |
| `[derived]` | Computed from observed values |
| `[inferred]` | Reasoned conclusion |

---

## 1. The artifact is a split, not a whole app

REA's Flutter triage returns `flutter_detected: false`, with the reason
*"No lib/\*/libapp.so was found"*. `[observed]`

The APK is the **base split of an Android App Bundle**:

- `res/xml/splits0.xml` lists splits, and they are **language configs only**
  (`config.af`, `config.am`, `config.ar`, … ) `[observed]`
- **Zero** `lib/` entries `[observed]`
- `assets/flutter_assets/` **is** present — AssetManifest, FontManifest,
  NOTICES.Z, fonts, images `[observed]`

So it is definitely Flutter, but the `libapp.so` holding the compiled Dart lives
in a per-ABI split (`config.arm64_v8a.apk` / `config.armeabi_v7a.apk`) that this
download did not include.

---

## 2. Platform facts

| Property | Value | Evidence |
| --- | --- | --- |
| Package | `com.buildcores.buildcores_flutter` | `[observed]` |
| Version | 2023.1.2 (34) | `[observed]` |
| minSdk / targetSdk / compileSdk | 25 / 32 / 33 | `[observed]` |
| Flutter embedding | **v2** (`flutterEmbedding` meta-data) | `[observed]` |
| Classes in DEX | **2,629** | `[observed]` |
| Resources | 333 | `[observed]` |
| Signature | v2 + v3 present, **v1 absent**; digest `sha512`, verified `matches_apk: true` | `[observed]` |
| Signer cert | `CN=Android, OU=Android, O=Google Inc., L=Mountain View` — **Aptoide re-signature**, not the original developer key | `[observed]` |

The certificate subject is Google's generic Android debug/release CN with a
30-year validity (2019→2049); that is the mirror's signature, so **integrity
here rests on the mirror, not the developer**. `[inferred]`

---

## 3. Permissions and components

```
android.permission.INTERNET
android.permission.ACCESS_NETWORK_STATE
android.permission.WAKE_LOCK
com.google.android.gms.permission.AD_ID
com.google.android.finsky.permission.BIND_GET_INSTALL_REFERRER_SERVICE
```

`[observed]`

Notably **absent**: location, storage, contacts, camera, phone. For a
price-aggregation app this is a minimal and appropriate set. `[observed]`

Components — 14 total, only one exported:

| Component | Exported | Notes |
| --- | --- | --- |
| `com.buildcores.buildcores_flutter.MainActivity` | **true** | launcher, `configChanges` covers everything |
| `io.flutter.plugins.urllauncher.WebViewActivity` | false | opens retailer links in-app |
| `io.flutter.plugins.imagepicker.ImagePickerFileProvider` | false | FileProvider |
| `io.flutter.plugins.share.ShareFileProvider` | false | FileProvider |
| `com.google.firebase.*` (init, discovery, measurement) | false | Firebase runtime |
| `androidx.lifecycle.ProcessLifecycleOwnerInitializer` | false | |
| `com.google.android.datatransport…TransportBackendDiscovery` | false | |

`<queries>` declares only an `https` `VIEW` intent — consistent with
`url_launcher` opening retailer pages. `[observed]`

---

## 4. Backend — the significant finding

`res/values/strings.xml` ships the full Firebase configuration:

| Key | Value |
| --- | --- |
| `firebase_database_url` | **https://buildcores.firebaseio.com** |
| `project_id` | `buildcores` |
| `google_app_id` | `1:351413753018:android:530e7250c09993c6` |
| `google_api_key` | `AIzaSyD15xLqlUnWH_m9mmQXUjMFhWHu29EX9Zw` |
| `gcm_defaultSenderId` | `351413753018` |
| `default_web_client_id` | `351413753018-f27n8a9k6rpk573tji2ac8ktonnpqplt.apps.googleusercontent.com` |
| `google_crash_reporting_api_key` | *(same as `google_api_key`)* |
| `com.crashlytics.android.build_id` | `d2938564-643e-446d-8de8-562584adf5e3` |

`[observed]`

**Architectural read:** the parts catalogue, pricing and benchmark data are
served from a **Firebase Realtime Database**. That single fact explains most of
the app's advertised behaviour:

- *"updated hourly with the latest pricing data"* → server-side writes to a
  Realtime Database the client reads
- *"completely usable when you're offline"* → Realtime Database's built-in
  on-device persistence and disk cache
- *"light data usage"* → incremental sync rather than full payloads
- *"builds stored locally"* → separate from the catalogue sync

`[inferred]`

**Firebase services wired in** (from `ComponentDiscoveryService` registrars):
Crashlytics, Performance Monitoring, Analytics, Remote Config, Storage, Core,
plus A/B Testing (`firebase-abt`) and Firebase Installations. `[observed]`

> **Security note, flagged not exploited.** A Firebase Realtime Database URL plus
> its API key is a classic exposure: the key is not a secret by itself, but if
> the database's security rules are permissive the whole dataset is readable by
> anyone. **I have not probed this endpoint and will not without your explicit
> confirmation that you own it or have permission.** If this is your app, the
> check is a single unauthenticated `GET` on
> `https://buildcores.firebaseio.com/.json` — say the word and I'll verify it.

---

## 5. AdMob present despite the "ad-free" claim

```
<meta-data android:name="com.google.android.gms.ads.APPLICATION_ID"
           android:value="ca-app-pub-4605343159451110~4303984663"/>
```

`[observed]`

The Play listing states *"BuildCores is entirely ad-free on Android."* The
AdMob application ID is declared, though no ad units or `AdActivity` are
present, and the permission list has no ad-related entries beyond `AD_ID`.
Most likely the SDK is initialised but no inventory is requested — a
dormant monetisation switch. `[inferred]`

---

## 6. Bundled data assets

### `assets/flutter_assets/assets/GPUBenchmarks.json`

**35 GPUs**, released 2016–2019. `[observed]`

Schema (13 fields):

```
vendor  architecture  releaseDate  coreCount  clockSpeed
ramSize  ramType  gamingPerformance  proPerformance
price  asin  image  amazonLink
```

Every `amazonLink` carries the affiliate tag **`tag=buildcoresx-20`** — the
Amazon Associates ID, hardcoded into a shipped asset. `[observed]`

Top entries by `gamingPerformance`: Titan RTX (14150), RTX 2080 Ti (13610),
RTX 2080 Super (11530), RTX 2080 (11002), RTX 2070 Super (10088).

Notably the 2023.1.2 changelog says *"Fixed GPU benchmarks not showing"* — this
bundled file is the fallback/offline benchmark source, and it freezes at 2019
hardware. `[inference]`

### `assets/flutter_assets/assets/buildcores.com.pem`

A leaf certificate for `CN=buildcores.com`, issued by **Let's Encrypt Authority
X3**, valid **5 Sep 2020 → 4 Dec 2020** — i.e. **expired**, and shipped
regardless. `[observed]`

This is **certificate pinning**: the app trusts this exact leaf (or its SPKI)
rather than the system trust store. Shipping an expired 2020 leaf is a real
robustness hazard — a pinned cert that can never be valid again means any
backend change requires an app update. `[inferred]`

### Other assets

- Retailer logo assets: `amazonwhite`, `newegglogo`, `walmartlogo`,
  `bbylogowhite` (Best Buy), `adoramalogowhite` — matching the changelog's
  *"Added Walmart and Adorama pricing"* `[observed]`
- Fonts: IBM Plex Sans (6 weights) + MaterialIcons + Font Awesome `[observed]`
- `packages/fluttertoast/assets/toastify.{css,js}` — the fluttertoast plugin
  in-app webview fallback `[observed]`
- `shaders/ink_sparkle.frag` — Flutter's Material ink effect `[observed]`

---

## 7. Plugins and third-party surface

From `AssetManifest.json`, component registrars and file-provider paths:

`url_launcher` · `image_picker` · `share_plus` (as `flutter_share`) ·
`fluttertoast` · `font_awesome_flutter` · `cupertino_icons` ·
`webview` (via urllauncher `WebViewActivity`) ·
Firebase (core, analytics, crashlytics, perf, remote-config, storage, abt,
installations) · Google Play Services (measurement, datatransport) ·
AndroidX (lifecycle, window extensions)

`[observed]`

`uses-library`: `org.apache.http.legacy`, `androidx.window.extensions`,
`androidx.window.sidecar` (foldable support). `[observed]`

---

## 8. The Dart payload — recovered

**The gap is closed.** The native split was obtained without an emulator and
without a browser: APKPure's XAPK endpoint accepts a plain `curl` once the
correct query parameters are supplied.

| Artifact | Bytes | SHA-256 |
|---|---|---|
| `buildcores-2023.1.2-apkpure.xapk` | 31,629,435 | `c3b03e14…` |
| `config.x86_64.apk` (the native split) | 20,095,461 | `02571ec8…` |
| `lib/x86_64/libapp.so` | 8,962,968 | `d73a359a…` |
| `lib/x86_64/libflutter.so` | 11,109,376 | `36982ce0…` |

**Integrity.** The XAPK's SHA-1 is `f13bce9d2ce76e07ee0f554461ae7fa69ae0437f` —
identical to the checksum APKPure publishes on its download page — and its size
matches the CDN's declared `full_size` exactly. The base APK inside it is
byte-identical to the Aptoide copy analysed earlier (MD5
`f1a67a11436c1797548946910a379cfd`), so two independent sources agree.

### Snapshot facts

- `flutter_detected: true`, ABI `x86_64`
- **Dart snapshot hash** `adb4292f3ec25074ca70abcd2d5c7251`
- Snapshot **kind 3** — the Dart 2.14 FullAOT layout REA can decode
- Engine build id `65bfcf19eebaa2a1389adba208f315356f92517b`
- Toolchain `Android (7155654, based on r399163b1) clang version 11.0.5`
  → NDK r22b, i.e. a **Flutter 2.8–2.10-era engine**, notably older than this
  build's Aug 2023 release date

| Section | Offset | Size | Kind |
|---|---|---|---|
| `_kDartVmSnapshotData` | 512 | 15,024 | 3 (header 13,578) |
| `_kDartIsolateSnapshotData` | 15,552 | 3,053,952 | 3 (header 2,447,967) |
| `_kDartVmSnapshotInstructions` | 3,080,192 | 20,096 | — |
| `_kDartIsolateSnapshotInstructions` | 3,100,288 | 5,846,704 | — |

### 129 Dart source files

The isolate string pool yields 129 `package:buildcores/…` paths — the complete
application structure:

| Subsystem | Files |
|---|---|
| `Screens/` | 56 |
| `Models/` | 35 |
| `Widgets/` | 11 |
| `Utilities/` | 9 |
| `Database/` | 9 |
| `API/` | 7 |
| `main.dart`, `src/app.dart` | 2 |

Full list: `evidence/dart_sources.txt`.

### What the structure reveals

- **There is a real backend, not just Firebase.** `src/API/` holds
  `buildcores_api.dart` plus `_auth`, `_users`, `_official_parts`,
  `_community_parts`, `_share_build` — and, notably, **`api_key_fetcher.dart`**.
  So the Firebase RTDB URL found in `strings.xml` is not the whole story; the app
  fetches API keys at runtime.
- **Auto Build** is `src/Database/AutomaticBuild/automatic_build.dart`, surfaced
  through `Screens/AutoBuild/generated_auto_build_view.dart` and
  `Screens/automatic_build_view.dart`.
- **A community-contributed parts system** exists — 20 files under
  `Screens/CommunityDatabase/`, including per-category forms (`cpu_form.dart`,
  `gpu_form.dart`, …), `private_part_form.dart`, and
  `community_part_status_view.dart`. Users submit parts; there is a moderation
  queue.
- **Retailer price fields are per-region**, confirmed by string tokens:
  `amazonUSPrice`, `amazonCAPrice`, `amazonPriceForRegion`,
  `amazonStockStatusUK`, `bestbuy_price`, `bestbuy_stock_status`,
  `adorama_availability`, `Walmart`.
- **Reddit export is first-class**: `Utilities/generate_reddit_output.dart`,
  alongside `generate_text_output.dart` and `generate_text_amazon_output.dart`.
- **Compatibility** is modelled in `Models/PartFilters/` (14 filter classes) and
  surfaced through `Screens/compatibility_filters_view.dart`.

### Notable loose end

The string pool contains `Uc8BN532Hg06EW1n8ksVjCWp3vSjdEkq` — a 32-character
opaque token, the shape of an API key or client secret. Its role is
**unconfirmed**; it may equally be a random identifier. I have not probed it, and
the same rule as the Firebase URL applies: nothing is sent to any endpoint
without your explicit say-so.

### Coverage limits of this pass

REA caps its projections: `package_uris` reported 500 of ≥1000 distinct URIs and
`class_like_tokens` 2000 of ≥4000. The 129 `buildcores` paths above were recovered
by pattern-matching raw ELF strings directly, which sidesteps the cap. A
Dart-version-specific cluster parser would be needed to go from *identifier-shaped
runs* to typed classes and functions.

The ABI analysed is **x86_64**. This is fine for string/URI recovery (the Dart
snapshot's data section is architecture-independent), but a native-code
disassembly pass would want `config.arm64_v8a.apk` instead.

## 9. Still open

- The `api_key_fetcher.dart` flow and the real API base URL — needs either a
  runtime trace (frida) or a static read of `buildcores_api.dart`'s constants
- Whether `Uc8BN532Hg06EW1n8ksVjCWp3vSjdEkq` is a live credential
- The 2,629 DEX classes, unexamined: `search-android-classes`,
  `inspect-android-class/method`, `project-android-application-graph`
- The current app is **BuildCores: 3D PC Builder** (`com.buildcores.buildcores`,
  2026.3.0, 170.8 MB) — a separate, much larger rebuild. Say the word and I'll
  start a second teardown against it.
