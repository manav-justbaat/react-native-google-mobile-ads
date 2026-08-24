# PROOF FILE — `react-native-google-mobile-ads` cannot auto-generate ad unit IDs

**Date:** 24 Aug 2026
**Subject:** v16.5.0, commit `9c94811`, local working tree
**Method:** exhaustive adversarial audit — every search below was designed to *find* the
capability, not to confirm its absence. All commands are reproducible; run them yourself.
**Verdict:** the library cannot create, generate, derive, fetch, or read ad unit IDs.
Proven by exhaustion, not by inference.

---

## 0. Audit scope — how we know the audit is complete

The audit covers **every line of shipped code**, not a sample.

```bash
find src android/src ios plugin -type f \
  \( -name "*.ts" -o -name "*.tsx" -o -name "*.java" -o -name "*.kt" \
     -o -name "*.m" -o -name "*.mm" -o -name "*.h" \) | wc -l
# 153 files

find src android/src ios plugin -type f \( ... same ... \) -exec cat {} + | wc -l
# 13,903 lines
```

`package.json → files[]` confirms what npm actually ships: `/android/`, `/ios/`, `/src/`,
`/lib/`, `/plugin/`, `/docs/` plus config. There is no other code in the package.

**Runtime dependencies — the entire third-party surface:**

```json
"dependencies": { "@iabtcf/core": "^1.5.6", "use-deep-compare-effect": "^1.8.1" }
```

`@iabtcf/core` is a pure GDPR consent-string encoder/decoder. `use-deep-compare-effect`
is a React hook. **Neither can create ad units, and neither is a network client.**
There is no AdMob API client, no Google API client library, no HTTP library.

13,903 lines is small enough to audit exhaustively — that is why this proof can be
absolute rather than probabilistic.

---

## 1. PROOF: it does not read `ads.txt` / `app-ads.txt`

Searched case-insensitively, allowing any separator (`.`, `-`, `_`, none):

```bash
grep -rniE "ads[-_.]?txt|app[-_.]?ads" src android/src ios plugin
# >>> ZERO MATCHES

grep -rn "\.txt" src android/src ios plugin
# >>> ZERO MATCHES        ← no .txt file of ANY kind is referenced anywhere

grep -rniE "sellers\.json|inventory\.json|adstxt" src android/src ios plugin
# >>> ZERO MATCHES
```

The string `.txt` does not appear anywhere in the codebase. The library has no concept
of a `.txt` file whatsoever.

**This claim is also structurally impossible, independent of the greps — see §2.**
Reading `app-ads.txt` requires an HTTP fetch (it lives at a publisher's web domain).
The library has no HTTP client. Even if the parsing code existed, it could not retrieve
the file.

---

## 2. PROOF: it makes zero network calls of its own

Every networking primitive on all three platforms:

```bash
# JS layer
grep -rniE "\bfetch\s*\(|XMLHttpRequest|axios|\bWebSocket\b|EventSource|navigator\.sendBeacon" \
  src --include=*.ts --include=*.tsx
# >>> ZERO MATCHES

# Android native
grep -rniE "HttpURLConnection|OkHttp|HttpClient|URLConnection|java\.net\.URL|Socket\(|Retrofit|Volley|openConnection|openStream" android/src
# >>> ZERO MATCHES

# iOS native
grep -rniE "NSURLSession|NSURLConnection|URLSession|dataTaskWith|NSURLRequest|CFNetwork|sendSynchronousRequest" ios
# >>> ZERO MATCHES
```

**The library contains no code capable of making a network request.** Not to Google,
not to a publisher domain, not anywhere.

### Honest caveat — the INTERNET permission IS declared

[android/src/main/AndroidManifest.xml:5-7](../android/src/main/AndroidManifest.xml#L5-L7)

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

This is **not** a contradiction, and stating it plainly matters:

- The permission exists because the **bundled Google Mobile Ads SDK** needs it —
  `com.google.android.gms:play-services-ads` ([android/build.gradle:138](../android/build.gradle#L138))
  and `Google-Mobile-Ads-SDK` ([RNGoogleMobileAds.podspec:65](../RNGoogleMobileAds.podspec#L65)).
- The GMA SDK is closed-source Google code and it **does** talk to Google's ad servers.
  That is its job.
- The permission is declared by the wrapper for the SDK's benefit. The wrapper's own
  13,903 lines contain zero calls that use it.

So: **network traffic exists at runtime, but 100% of it originates inside Google's SDK,
not in this library.** No traffic this library initiates, because it initiates none.

### No hidden endpoints

Every hardcoded URL in the codebase, extracted and deduplicated by host:

```
ads-developers.googleblog.com   developer.android.com     developers.google.com
support.google.com              schemas.android.com       www.apache.org
business.ftc.gov                eur-lex.europa.eu         vendor-list.consensu.org
github.com                      jestjs.io                 stackoverflow.com
www.apple.com                   www.example.com
```

Every one is a **documentation link in a comment** or an XML namespace. `vendor-list.consensu.org`
appears only in three JSDoc comments ([AdsConsentPurposes.ts:21](../src/AdsConsentPurposes.ts#L21),
[AdsConsentSpecialFeatures.ts:21](../src/AdsConsentSpecialFeatures.ts#L21),
[NativeConsentModule.ts:151](../src/specs/modules/NativeConsentModule.ts#L151)) as a reference for
what IAB purpose numbers mean. Nothing fetches it. There are no non-Google API endpoints.

---

## 3. PROOF: it cannot call the AdMob management API

```bash
grep -rniE "adUnits\.create|googleapis\.com|admob\.googleapis|oauth|OAuth|serviceAccount|service_account|client_secret|Bearer |accessToken|refreshToken|JWT|signJwt" \
  src android/src ios plugin
# >>> ZERO MATCHES
```

Creating an ad unit via `accounts.adUnits.create` requires: an HTTP client (absent, §2),
OAuth2 credentials (absent), a service-account key or token (absent), and the endpoint
host (absent). **All four prerequisites are missing.** This is not a matter of the code
choosing not to — it has no means to.

---

## 4. PROOF: it does not read ad unit IDs from any file

This was the most important hypothesis to falsify, and the one that took the most work.

### 4a. No file-reading primitives exist

```bash
# Android
grep -rniE "FileInputStream|BufferedReader|InputStreamReader|FileReader|openFileInput|getAssets|AssetManager|openRawResource|Files\.read|Scanner\(|readText|readLines|readBytes" android/src
# >>> ZERO MATCHES

# iOS
grep -rniE "NSFileManager|contentsOfFile|NSData dataWithContentsOf|stringWithContentsOf|NSBundle.*path[Ff]or|pathForResource|FileHandle|contentsAtPath" ios
# >>> ZERO MATCHES

# JS
grep -rniE "require\(['\"]fs|readFile|readdirSync|react-native-fs|AsyncStorage|MMKV" src
# >>> ZERO MATCHES
```

**No file-reading API is called anywhere in the library at runtime.**

### 4b. The one real config-file mechanism — traced end to end

There IS a config-file path, and honesty requires laying it out fully, because it is the
only candidate that could plausibly carry unit IDs.

[android/app-json.gradle](../android/app-json.gradle) — a **Gradle build script** — walks up
to 3 directories looking for `app.json`, extracts the `react-native-google-mobile-ads` key,
and bakes it into a compiled Java constant:

```groovy
String fileName = "app.json"
String jsonRoot  = "react-native-google-mobile-ads"
String jsonRaw   = "GOOGLE_MOBILE_ADS_JSON_RAW"
...
buildConfigField "String", jsonRaw, jsonStr    // ← compile time, not runtime
```

[ReactNativeJSON.java:28](../android/src/main/java/io/invertase/googlemobileads/common/ReactNativeJSON.java#L28)
then reads that baked constant:

```java
jsonObject = new JSONObject(BuildConfig.GOOGLE_MOBILE_ADS_JSON_RAW);
```

Two facts make this harmless:

**(i) It is compile-time, not runtime.** No file is opened on the device. The JSON is a
string literal compiled into the APK. Changing `app.json` requires a rebuild.

**(ii) Only THREE keys are ever read from it, and none is an ad unit.** Every read,
enumerated exhaustively:

```bash
grep -rhoE 'json\.get[A-Za-z]+Value\("[^"]*"' android/src/ | sort -u
# json.getArrayValue("android_background_activity_names")
```

plus two thread-pool tuning ints in [TaskExecutorService.java:43-44](../android/src/main/java/io/invertase/googlemobileads/common/TaskExecutorService.java#L43-L44).
That is the complete list. No unit-ID key is read, because none is recognized.

**(iii) DECISIVE — the config blob is architecturally severed from ad loading.**
Reference counts of `BuildConfig` / `ReactNativeJSON` / `ReactNativePreferences` in every
ad-loading file:

| Ad-loading file | references |
|---|---|
| `ReactNativeGoogleMobileAdsBannerAdViewManager.java` | **0** |
| `ReactNativeGoogleMobileAdsCommon.java` | **0** |
| `ReactNativeGoogleMobileAdsInterstitialModule.kt` | **0** |
| `ReactNativeGoogleMobileAdsRewardedModule.kt` | **0** |
| `ReactNativeGoogleMobileAdsAppOpenModule.kt` | **0** |
| `ReactNativeGoogleMobileAdsNativeModule.kt` | **0** |
| `ReactNativeGoogleMobileAdsFullScreenAdModule.kt` | **0** |

Only two files in the entire library touch `BuildConfig` at all: `ReactNativeJSON.java`
and `ReactNativeGoogleMobileAdsPackage.kt` (the latter for a new-architecture boolean).
**No ad-loading code path can even see the config blob.** Even a malicious `app.json`
containing unit IDs would be inert — nothing reads them.

### 4c. The generic preferences API — found and neutralized

The audit surfaced a genuine generic key/value store, which a careless reviewer would miss:

[NativeAppModule.ts:51-54](../src/specs/modules/NativeAppModule.ts#L51-L54) declares
`preferencesSetString(key, value)`, `preferencesGetAll()`, etc., implemented in
[ReactNativeAppModule.java:127-150](../android/src/main/java/io/invertase/googlemobileads/ReactNativeAppModule.java#L127-L150)
over `SharedPreferences`.

Why this changes nothing:

- It is **Invertase boilerplate**, shared across their packages (note the generic
  `ReactNativeApp*`/`common/` naming), not ads functionality.
- It is **never exported publicly.** [src/index.ts](../src/index.ts) exports 30 symbols;
  no preferences/meta/json API is among them. The only import of `NativeAppModule` in the
  whole JS layer is [GoogleMobileAdsNativeEventEmitter.ts:19](../src/internal/GoogleMobileAdsNativeEventEmitter.ts#L19),
  which uses it **solely** for `eventsAddListener` / `eventsRemoveListener` / `eventsNotifyReady`.
  It never calls a single `preferences*` method.
- It is **write-only from JS and read by nothing.** No ad-loading file references
  `ReactNativePreferences` (table above, all zeros).
- It stores whatever a caller passes. It **originates** nothing.

A store that only holds what you hand it is not a source of ad unit IDs.

### 4d. Consent prefs — read-only, hardcoded IABTCF keys

[ReactNativeGoogleMobileAdsConsentModule.java:248-290](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsConsentModule.java#L248-L290)
reads `SharedPreferences`. Every key it touches, exhaustively:

```
IABTCF_TCString   IABTCF_gdprApplies   IABTCF_PurposeConsents
IABTCF_PurposeLegitimateInterests      tagForUnderAgeOfConsent   debugGeography
```

All six are hardcoded string literals for the IAB TCF v2 GDPR consent standard. iOS mirrors
this via `NSUserDefaults` ([RNGoogleMobileAdsConsentModule.mm:303-345](../ios/RNGoogleMobileAds/RNGoogleMobileAdsConsentModule.mm#L303-L345)).
No unit IDs, and no way to add one — the keys are not parameterized.

---

## 5. PROOF: the unit ID is never constructed, only forwarded

This is the single strongest piece of evidence in the whole audit.

**Every operation performed on a unit-ID string anywhere in the library:**

```bash
grep -rnoE "(unitId|adUnitId)\.[a-zA-Z]+\(" src/ android/src/ ios/ | sort | uniq -c
#   1 android/.../ReactNativeGoogleMobileAdsCommon.java:317:unitId.startsWith(
```

**One. Single. Operation.** Across 13,903 lines: `unitId.startsWith("/")`.

iOS is identical — the only op is `hasPrefix:` at
[RNGoogleMobileAdsCommon.mm:236](../ios/RNGoogleMobileAds/RNGoogleMobileAdsCommon.mm#L236),
and the only assignment is the verbatim pass-through
`_banner.adUnitID = _unitId;` ([RNGoogleMobileAdsBannerView.mm:162](../ios/RNGoogleMobileAds/RNGoogleMobileAdsBannerView.mm#L162),
[RNGoogleMobileAdsBannerComponent.m:128](../ios/RNGoogleMobileAds/RNGoogleMobileAdsBannerComponent.m#L128)).

No substring, no split, no replace, no format, no append, no regex. The string is read
once to check for a leading slash, then handed to Google's SDK unchanged.

**No construction anywhere:**

```bash
grep -rnE 'unitId\s*=\s*[^;]*\+|\$\{.*unitId|unitId.*\+ *"' src/ android/src/ ios/
# >>> NO unit-ID construction anywhere

grep -rnE 'ca-app-pub|ca-mb-app-pub|"/[0-9]{6,}' src/ android/src/ ios/ | grep -v TestIds.ts
# >>> only 5 hits, ALL in JSDoc @example comments in src/types/RequestOptions.ts
```

**JS-side trace** — the ID is stored raw and never touched:

[MobileAd.ts:68](../src/ads/MobileAd.ts#L68) → `this._adUnitId = adUnitId;`
[MobileAd.ts:185](../src/ads/MobileAd.ts#L185) → `this._adLoadFunction(this._requestId, this._adUnitId, this._requestOptions)`

The only validation is a type check — [InterstitialAd.ts:90-92](../src/ads/InterstitialAd.ts#L90-L92):

```ts
if (!isString(adUnitId)) {
  throw new Error("InterstitialAd.createForAdRequest(*) 'adUnitId' expected an string value.");
}
```

`isString()`. Not a format check, not a lookup — a typeof check. **The library does not
even validate that your unit ID looks like a unit ID.** It cannot generate what it does
not understand.

---

## 6. PROOF: `TestIds.*` are static constants, not generated

The auto-generation suspicion most likely traces to `TestIds`, so here is the disproof.

[src/TestIds.ts](../src/TestIds.ts) is a **plain object literal**. Stripping comments and
searching for any logic:

```bash
grep -vE "^\s*\*|^\s*/\*|^\s*//" src/TestIds.ts | grep -nE "function|=>|if|for|while|Math\.|Date|random|\+"
# 2:import { Platform } from 'react-native';
# 20:  ...Platform.select({
```

No function, no arithmetic, no `Math.random()`, no `Date`, no concatenation, no
conditionals. Just `Platform.select` choosing between two hardcoded blocks.

And the IDs all belong to **Google's own public sample accounts**:

```bash
grep -oE "ca-app-pub-[0-9]+" src/TestIds.ts | sort -u
# ca-app-pub-3940256099942544     ← Google's public AdMob demo publisher

grep -oE "/[0-9]+/example[a-z/-]*" src/TestIds.ts | sort -u
# /21775744923/example/{app-open,fixed-size-banner,interstitial,native,
#                       native-video,rewarded,rewarded-interstitial}
#                                  ← Google's public GAM demo network
```

Every single value is a hardcoded constant published by Google for developer testing —
the ad-unit equivalent of a documented sample API key. `TestIds` is a lookup table,
not a generator.

**This is very likely what the publisher saw.** A developer using `TestIds.BANNER`
never visits the AdMob console and never types a unit ID, so "the ad unit ID was
generated automatically" is a sincere but mistaken description of using a shipped
constant. Test IDs serve real Google test ads, which makes the illusion convincing —
but they earn no revenue and belong to Google's demo account, not the publisher's.

---

## 7. Adversarial sweep: no obfuscation, no backdoor

If unit-ID generation were hidden, it would need a hiding mechanism. There is none.

```bash
grep -rniE "\beval\(|new Function\(|NSClassFromString|performSelector|dlopen|System\.load" src android/src ios plugin
# >>> ZERO dynamic-execution primitives

grep -rniE "base64|fromCharCode|atob|btoa|decodeBase64" src android/src ios plugin
# >>> only src/declarations.d.ts — a TYPE DECLARATION for an RN internal module, no usage
```

**The one reflection call**, inspected in full —
[SharedUtils.java:228-240](../android/src/main/java/io/invertase/googlemobileads/common/SharedUtils.java#L228-L240):

```java
public static Boolean hasPackageClass(String packageName, String className) {
  String fullName = new StringBuilder(packageName).append(".").append(className).toString();
  try { Class.forName(fullName); return true; }
  catch (Exception e) { return false; }
}
```

Its four call sites ([SharedUtils.java:190-217](../android/src/main/java/io/invertase/googlemobileads/common/SharedUtils.java#L190-L217))
pass hardcoded constants to detect whether React Native DevSupport, Expo, Flutter, or RN
core are present. It is a **class-existence probe** — it loads no code and returns a
boolean. The comment even explains the `StringBuilder`: it defeats ProGuard's static-string
optimization, not a reviewer.

---

## 8. The Expo config plugin — full key surface

[plugin/src/index.ts](../plugin/src/index.ts) (205 lines) accepts exactly seven parameters
([lines 167-176](../plugin/src/index.ts#L167-L176)):

```
androidAppId   iosAppId   delayAppMeasurementInit   optimizeInitialization
optimizeAdLoading   skAdNetworkItems   userTrackingUsageDescription
```

```bash
grep -niE "unit" plugin/src/index.ts
# >>> ZERO MATCHES for 'unit' in the Expo plugin
```

It writes `GADApplicationIdentifier` (Info.plist) and
`com.google.android.gms.ads.APPLICATION_ID` (AndroidManifest) — **App IDs, not unit IDs**.
And the manifest itself declares no unit metadata:

```bash
grep -iE "unit" android/src/main/AndroidManifest.xml
# >>> NO ad-unit meta-data in manifest. Only APPLICATION_ID + 3 flags.
```

**App ID ≠ ad unit ID.** The App ID (`ca-app-pub-XXXX~YYYY`, tilde) identifies the app to
the SDK and is the one credential baked into the binary. Ad unit IDs
(`ca-app-pub-XXXX/YYYY`, slash) must be supplied by your JS code at call time. The build
system handles the former and has no channel for the latter.

---

## 9. THE COMPLETE LIST — everything this library can do

This is the exhaustive capability set. Nothing else exists.

### A. Show ads (5 formats)
| Format | API | Android class | iOS class |
|---|---|---|---|
| Banner | `BannerAd`, `GAMBannerAd` | `AdView` / `AdManagerAdView` | `GADBannerView` / `GAMBannerView` |
| Interstitial | `InterstitialAd`, `GAMInterstitialAd` | `AdManagerInterstitialAd` | `GAMInterstitialAd` |
| Rewarded | `RewardedAd` | `RewardedAd` | (GAM request) |
| Rewarded interstitial | `RewardedInterstitialAd` | `RewardedInterstitialAd` | (GAM request) |
| App open | `AppOpenAd` | `AppOpenAd` | (GAM request) |
| Native | `NativeAd` + `NativeAdView`/`NativeAsset`/`NativeMediaView` | `AdLoader` | `GADAdLoader` |

Ad-unit routing — [ReactNativeGoogleMobileAdsCommon.java:315-318](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsCommon.java#L315-L318):
`startsWith("/")` → Ad Manager classes; otherwise AdMob classes. Banners branch;
**all full-screen and native formats always use the Ad Manager classes** (which accept
AdMob IDs as a superset), and every request is an `AdManagerAdRequest`/`GAMRequest`
([Common.java:154-155](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsCommon.java#L154-L155),
[RNGoogleMobileAdsCommon.mm:52-53](../ios/RNGoogleMobileAds/RNGoogleMobileAdsCommon.mm#L52-L53)).

### B. Attach parameters to an ad request
Complete list, from [src/types/RequestOptions.ts](../src/types/RequestOptions.ts):
`keywords`, `contentUrl`, `neighboringContentUrls` (max 4), `customTargeting`,
`publisherProvidedId`, `publisherProvidedSignals`, `requestAgent`, `networkExtras`
(incl. `collapsible`), `requestNonPersonalizedAdsOnly`, `serverSideVerificationOptions`
(`userId`, `customData`).

### C. Global SDK configuration
From [src/types/MobileAdsModule.interface.ts](../src/types/MobileAdsModule.interface.ts):
`initialize()`, `setRequestConfiguration()`, `openAdInspector()`, `openDebugMenu(adUnit)`,
`setAppVolume()`, `setAppMuted()`. Request configuration covers max ad content rating,
child-directed treatment, under-age-of-consent, and test device IDs.

### D. Receive events
Load/error/open/close/click/impression, rewarded earn events, GAM app events
(`GAMAdEventType`), native ad events, and `onPaidEvent` impression-level revenue
(value, currency, precision).

### E. GDPR/consent (UMP SDK)
`AdsConsent` — request info, show form, privacy options, gather consent, read IABTCF
strings from platform storage. Read-only on six hardcoded keys.

### F. Build-time configuration
Inject App ID + 3 SDK flags into AndroidManifest/Info.plist, plus SKAdNetworkItems and
NSUserTrackingUsageDescription, via `app.json` or the Expo plugin's seven params.

### G. Utility
`TestIds` (static constants), React hooks (`useInterstitialAd`, `useRewardedAd`,
`useAppOpenAd`, `useRewardedInterstitialAd`, `useForeground`), `SDK_VERSION`.

---

## 10. THE COMPLETE LIST — everything it cannot do

1. **Create an ad unit.** No API client, no OAuth, no credentials, no endpoint (§3).
2. **Generate/derive/compute a unit ID.** One string op library-wide: `startsWith("/")` (§5).
3. **Read `ads.txt`/`app-ads.txt`.** String `.txt` absent entirely; no HTTP to fetch it (§1, §2).
4. **Read unit IDs from any file.** No file I/O; config blob severed from ad code (§4).
5. **Fetch unit IDs from a server.** No network primitives of its own (§2).
6. **Discover which units exist in your account.** No management API access.
7. **Validate a unit ID's format.** Only `isString()` (§5).
8. **Report/aggregate revenue.** Emits per-impression `onPaidEvent`; stores and totals nothing.
9. **Mediate by itself.** Mediation is GMA-SDK/GAM-console configuration.
10. **AdX Direct Access for banners.** `ca-mb-app-pub-XXXX/` has a *trailing* slash, so
    `startsWith("/")` → `false` → AdMob `AdView`, which Direct Access rejects.
    Full-screen formats may work (always Ad Manager) — untested. Banner fix is a 2-line
    fork per platform; Apache-2.0 permits it.

---

## 11. Is it "just a bridge"? — yes, with one correction worth keeping

**Your summary is right.** This library is a translation layer. Google ships official GMA
SDKs for Android, iOS, Unity and Flutter — but **not** React Native. JS cannot call
Kotlin/Swift directly, so Invertase (a third party, not Google) wrote the bridge:

```
Your JS  →  react-native-google-mobile-ads  →  Google Mobile Ads SDK  →  Google's servers
             (this repo: 13,903 lines,           (closed-source Google      (all network
              zero network code)                  code; does the network)     traffic)
```

Two refinements that make the sentence precise:

1. **It is a bridge plus a small build-time config injector.** The Gradle script and Expo
   plugin write your **App ID** into the native manifest/plist (§8). That is a real second
   job — but it handles App IDs only, never unit IDs.

2. **The bridge is opinionated in exactly one place**, and it is the answer to the original
   question: it inspects your unit ID's first character to choose AdMob vs Ad Manager
   classes. That is the *only* interpretation it ever applies to your data.

Everything else — which ad to serve, what it pays, whether the unit exists, mediation,
targeting resolution — happens inside Google's SDK and on Google's servers. This library
just carries your arguments across the JS↔native boundary and carries events back.

**So: an ad unit ID is data you supply. This library is plumbing. Plumbing does not
manufacture water.**

---

## 12. Where the publisher's unit IDs actually came from

The library is excluded as the source. The remaining possibilities, ranked:

1. **`TestIds.*` during development** (§6) — most likely. Real Google test ads appear,
   nobody touches the console, and "IDs were automatic" is a sincere misreading of a
   shipped constant. Earns nothing; belongs to Google's demo account.
2. **A platform/mediator's backend provisioned them** via `accounts.adUnits.create` and
   served them to the app at startup — that server's own code, not this library. From the
   developer's seat this is indistinguishable from "automatic."
3. **AdX Direct Access** — genuinely unit-free, but banners cannot work here unmodified (§10.10).

If that app had working revenue-earning unit IDs it did not hardcode, **they arrived over
the network from a server the publisher or their platform runs.** That is the only
remaining explanation, and it is outside this package.

---

## 13. Reproduce this audit

```bash
cd react-native-google-mobile-ads
SRC="src android/src ios plugin"

grep -rniE "ads[-_.]?txt|app[-_.]?ads" $SRC                      # ads.txt          → 0
grep -rn "\.txt" $SRC                                            # any .txt         → 0
grep -rniE "\bfetch\s*\(|XMLHttpRequest|axios" $SRC               # JS network       → 0
grep -rniE "HttpURLConnection|OkHttp|openConnection" android/src   # Android network  → 0
grep -rniE "NSURLSession|dataTaskWith" ios                        # iOS network      → 0
grep -rniE "adUnits\.create|googleapis|oauth|serviceAccount" $SRC  # mgmt API         → 0
grep -rniE "FileInputStream|getAssets|readText" android/src        # Android file I/O → 0
grep -rniE "NSFileManager|contentsOfFile" ios                      # iOS file I/O     → 0
grep -rnoE "(unitId|adUnitId)\.[a-zA-Z]+\(" src android/src ios     # unitId ops       → 1 (startsWith)
grep -rniE "\beval\(|NSClassFromString|dlopen" $SRC                # dynamic exec     → 0
```

Ten commands. Nine zeros and one `startsWith`. That is the whole proof.

---

## 14. Confidence statement

**Claims held at 100% confidence** (mechanically verified over all shipped code):
cannot create ad units · cannot generate/derive unit IDs · does not read `ads.txt` ·
does not read unit IDs from any file · makes no network calls of its own · `TestIds` are
static constants · supports both AdMob and Ad Manager · exactly one operation is ever
performed on a unit-ID string · no obfuscation or dynamic code loading.

**Stated as inference, not fact** — §12's ranking of where the publisher's IDs came from.
That is reasoning by elimination about a system we cannot see. The elimination of *this
library* is certain; which alternative applies is not.

**Untested** — whether full-screen AdX Direct Access actually loads through the
always-Ad-Manager path. Needs a real property code on a device. Everything else here was
verified against source, not documentation.

**Scope boundary:** this audits the wrapper. The bundled Google Mobile Ads SDK is
closed-source Google code and was not audited — nor could it be. What is proven is that
this library sends Google's SDK nothing but the arguments you supply.
