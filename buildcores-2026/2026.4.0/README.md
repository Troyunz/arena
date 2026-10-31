# BuildCores 2026.4.0 — `com.buildcores.buildcores`

The current app. **Not Flutter** — this is React Native (Expo) with Google
Filament for 3D.

| Item | Value |
|---|---|
| Version | 2026.4.0 (versionCode 113), 14 Oct 2026 |
| minSdk / targetSdk | 29 / 36 |
| ABI | arm64-v8a |
| Permissions | 35 (incl. CAMERA, RECORD_AUDIO, BILLING, POST_NOTIFICATIONS) |
| Signature | `bc7616efb4436cc9c0b3aaae020e788e33bc5fad` (APKPure "Developer Verified") |

## Files

| File | Bytes | Why it matters |
|---|---|---|
| `index.android.bundle` | 15,957,996 | **Hermes bytecode** — the entire app logic. Magic `c6 1f bc 03` |
| `config.arm64_v8a.apk` | 50,445,624 | 38 native libs: `libreactnative.so`, `libhermesvm.so`, `libreact-native-filament.so`, `librnskia.so` |
| `manifest.json` | 2,092 | XAPK manifest |

The 164 MB base APK is **not** archived — it exceeds GitHub's 100 MB per-file
limit. It is mostly `res/` (82.8 MB of 3D assets) plus `assets/` (19.1 MB).
Re-fetch with:

```bash
UA='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36'
REF='https://apkpure.com/buildcores-3d-part-picker-pc/com.buildcores.buildcores/download'
curl -sSL -A "$UA" -e "$REF" -o buildcores-2026.4.0.xapk \
  'https://d.apkpure.com/b/XAPK/com.buildcores.buildcores?version=latest'
```

## Key findings

- **Framework change**: Flutter (2023) → React Native + Expo + Hermes (2026).
  Google Filament (`libreact-native-filament.so`) powers the 3D builder.
- **Firebase RTDB is still there**: `https://buildcores.firebaseio.com`, plus
  Firebase Auth (Identity Toolkit `/v1/accounts:*`).
- **A real API now exists**: `https://www.api.buildcores.com` with ~25 routes,
  including **admin endpoints** (`/api/official/database/admin/upload-3d-models`,
  `/api/official/database/admin/upload-part-image-file`) and an analytics
  pipeline (`/api/events/collectAnalyticsEvents`).
- **Monetisation flipped**: 2023 was "entirely ad-free". Now RevenueCat powers
  subscriptions — `rc_monthly`, `rc_annual`, `rc_two_month`, `rc_three_month`,
  `rc_six_month`, `rc_lifetime` — and the app declares `BILLING`.
- **Staging backend leaked in the bundle**:
  `https://buildcores-next-beta-o46sp.ondigitalocean.app`
- 3D assets are fetched from `https://static.buildcores.com/3d-assets/ghost-parts/`
  (`atx_ghost_mobo.glb`, `ghost_ddr5.glb`, `ghost_radiator_360.glb`, …).

See `evidence/network-surface.txt` for the full host/route inventory.

## Caveat

Third-party application binaries, archived for analysis only. A Google API key is
visible in the string table; it has **not** been used or probed.
