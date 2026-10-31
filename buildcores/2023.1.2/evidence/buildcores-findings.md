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

## The gap

**The Dart application code is not in this artifact.** Everything above is the
native shell, the resource layer and the bundled data. The actual app
logic — compatibility rules, Auto Build, part models, pricing
normalisation — is compiled AOT into `libapp.so`, which lives in the ABI split.

To close it I need one of:

1. **`config.arm64_v8a.apk`** (~20 MB) from the same version — the split
   carrying `lib/*/libapp.so`
2. A **universal APK** — APKMirror lists 64.8 MB
   (`arm64-v8a + armeabi-v7a + x86_64`, nodpi), but its download flow is
   JS-nonce gated and I could not complete it
3. An **XAPK** from APKPure (a zip of base + splits) — their CDN returned 403
4. If you have the app on a phone: **APK Extractor / Share APK** merges the
   splits into one installable APK — attach that

With `libapp.so` in hand, the REA passes that unlock are:

```bash
rea identify-flutter-build <apk>     # Dart SDK version from snapshot hash
rea inspect-dart-aot <apk>           # string pool, class tokens, package URIs
```

`inspect-dart-aot` is where the app's strings surface: API paths, part-category
names, compatibility messages, and the Auto Build logic.

## What I could still do without it

- `rea search-android-classes` / `inspect-android-class` / `inspect-android-method`
  over the 2,629 DEX classes — the Kotlin/Java embedding and plugin layer
- `rea project-android-application-graph` — full application graph
- `rea decode-android-resources` — full decoded resource tree (already done
  directly with apktool at `extracted/decoded/`)
- `rea inspect-apk-signature` — already completed
