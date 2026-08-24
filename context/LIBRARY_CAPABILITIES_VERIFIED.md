# `react-native-google-mobile-ads` — Verified Capabilities

**Date:** 24 Aug 2026
**Method:** direct source read of v16.5.0 working tree (commit `9c94811`), not docs
**Purpose:** settle what this library can and cannot do, for the AdMob/AdX ad-unit research

---

## Headline: the previous handoff's central assumption was WRONG

`ADMOB_ADX_HANDOFF.md` line 123 said:

> Docs mention **no** Ad Manager / GAM / AdX support — AdMob only.

**This is false.** The library has first-class Google Ad Manager support, on both platforms,
for every ad format. The docs simply under-advertise it. The handoff's own advice — "docs
often lag code, check the source" — was correct, and the source says GAM is supported.

---

## THE KEY QUESTION — ANSWERED

> Does the Android native bridge instantiate `AdView` or `AdManagerAdView`?

**Both. It switches at runtime based on the shape of the `unitId` string you pass.**

[ReactNativeGoogleMobileAdsBannerAdViewManager.java:221-224](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsBannerAdViewManager.java#L221-L224)

```java
BaseAdView adView =
    ReactNativeGoogleMobileAdsCommon.isAdManagerUnit(reactViewGroup.getUnitId())
        ? new AdManagerAdView(currentActivity)
        : new AdView(currentActivity);
```

The discriminator — [ReactNativeGoogleMobileAdsCommon.java:315-318](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsCommon.java#L315-L318)

```java
public static boolean isAdManagerUnit(String unitId) {
  if (unitId == null) return false;
  return unitId.startsWith("/");
}
```

iOS is identical in structure — [RNGoogleMobileAdsCommon.mm:232-237](../ios/RNGoogleMobileAds/RNGoogleMobileAdsCommon.mm#L232-L237)

```objc
+ (BOOL)isAdManagerUnit:(NSString *)unitId {
  if (unitId == nil) return NO;
  return [unitId hasPrefix:@"/"];
}
```

…driving the banner branch at [RNGoogleMobileAdsBannerView.mm:130-137](../ios/RNGoogleMobileAds/RNGoogleMobileAdsBannerView.mm#L130-L137)
(`GAMBannerView` vs `GADBannerView`).

**So: one leading slash is the entire GAM/AdMob switch.** `/22846411849/JBM_AVD_VN_banner`
already routes to `AdManagerAdView` today, with no fork required.

### Full-screen formats go further — they are ALWAYS Ad Manager

Interstitial, rewarded, rewarded-interstitial, app-open and native do not branch at all.
They unconditionally use the Ad Manager classes, which accept AdMob unit IDs as a superset:

- [ReactNativeGoogleMobileAdsInterstitialModule.kt:56](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsInterstitialModule.kt#L56) — `AdManagerInterstitialAd.load(...)`
- [ReactNativeGoogleMobileAdsCommon.java:154-155](../android/src/main/java/io/invertase/googlemobileads/ReactNativeGoogleMobileAdsCommon.java#L154-L155) — **every** ad request is an `AdManagerAdRequest.Builder`
- [RNGoogleMobileAdsInterstitialModule.mm:42](../ios/RNGoogleMobileAds/RNGoogleMobileAdsInterstitialModule.mm#L42) — `[GAMInterstitialAd loadWithAdManagerAdUnitID:...]`
- [RNGoogleMobileAdsCommon.mm:52-53](../ios/RNGoogleMobileAds/RNGoogleMobileAdsCommon.mm#L52-L53) — every request is a `GAMRequest`

There is no AdMob-only code path for full-screen ads in this library at all.

### Corroborating evidence: shipped GAM test IDs

[src/TestIds.ts:27-34](../src/TestIds.ts#L27-L34) ships Google's official GAM sample units:

```
GAM_APP_OPEN:              /21775744923/example/app-open
GAM_BANNER:                /21775744923/example/fixed-size-banner
GAM_INTERSTITIAL:          /21775744923/example/interstitial
GAM_REWARDED:              /21775744923/example/rewarded
GAM_REWARDED_INTERSTITIAL: /21775744923/example/rewarded-interstitial
GAM_NATIVE:                /21775744923/example/native
GAM_NATIVE_VIDEO:          /21775744923/example/native-video
```

Nobody ships and tests seven GAM unit IDs for a format they don't support.

---

## GAM-specific features actually exposed to JS

| Feature | Where | GAM-only? |
|---|---|---|
| `GAMBannerAd` component (multi-size `sizes[]`) | [src/index.ts:52](../src/index.ts#L52), [BannerAdProps.ts:142-148](../src/types/BannerAdProps.ts#L142-L148) | yes |
| `GAMInterstitialAd` | [src/index.ts:53](../src/index.ts#L53) | yes |
| `GAMAdEventType` / `onAppEvent` (creative→app messaging) | [src/index.ts:39](../src/index.ts#L39), [BannerAdProps.ts:183](../src/types/BannerAdProps.ts#L183) | yes |
| `manualImpressionsEnabled` + `recordManualImpression()` | [BannerAdProps.ts:178](../src/types/BannerAdProps.ts#L178) | yes |
| `customTargeting` (key-values → line item targeting) | [RequestOptions.ts:119](../src/types/RequestOptions.ts#L119) | yes |
| `publisherProvidedId` (PPID) | [RequestOptions.ts:148](../src/types/RequestOptions.ts#L148) | yes |
| `contentUrl` / `neighboringContentUrls` | [RequestOptions.ts:87](../src/types/RequestOptions.ts#L87) | both |
| `onPaidEvent` impression-level revenue | banner manager + full-screen modules | both |

`customTargeting` is the important one for our business model — see below.

---

## What the library CANNOT do

Verified by exhaustive grep across `src/`, `android/src/`, `ios/RNGoogleMobileAds/`:

1. **It cannot create ad units.** No AdMob API client, no OAuth, no service-account
   handling, no `accounts.adUnits.create` call. Nothing.
2. **It does not read `ads.txt` or `app-ads.txt`.** Zero matches for either string
   anywhere in the codebase. This part of the publisher's story is simply not true.
3. **It makes no network calls of its own.** No `fetch`, no HTTP client, no remote
   config, no phone-home. The only outbound traffic is the Google Mobile Ads SDK
   talking to Google's ad servers.
4. **It does not generate, derive, or infer unit IDs.** The `unitId` you pass is
   forwarded verbatim to the native SDK. The only inspection ever performed on that
   string is `startsWith("/")`.
5. **It holds no credentials on device** beyond the App ID baked into the manifest /
   `Info.plist` via the config plugin (`android_app_id` / `ios_app_id` in `app.json`).

### Therefore: the publisher's "automatic ad unit ID" claim

The library is not the explanation. Three plausible real explanations, most likely first:

1. **The publisher was onboarded through a platform/mediator** whose backend created
   units via `accounts.adUnits.create` and served the IDs to the app at startup. From
   the developer's chair this looks exactly like "I never created an ad unit."
   The IDs came down the wire from their own server, not from this library.
2. **They used `TestIds.*`** during development and mistook Google's built-in sample
   units for auto-generation. This is very common and fits the story well.
3. **They used AdX Direct Access** (`ca-mb-app-pub-XXXX/`) — genuinely no ad unit.
   But see the caveat below: this library would *reject* that path for banners.

Nothing in this package generates an ID. If the publisher's app had unit IDs it
didn't hardcode, they arrived from a server the publisher (or their platform) runs.

---

## AdX Direct Access through this library — PARTIAL, and broken for banners

Direct Access format is `ca-mb-app-pub-5629679302779023/` — **trailing** slash, no
leading slash.

Run that through `isAdManagerUnit()`:

```
"ca-mb-app-pub-5629679302779023/".startsWith("/")   →  false
```

**Banners therefore get plain `AdView`/`GADBannerView` — the AdMob class.** Google's
Direct Access docs require `AdManagerAdView` + `AdManagerAdRequest`. So banner Direct
Access **cannot work** on this library unmodified. It would fail, most likely with the
documented `Invalid Request. Cannot determine request type.` error.

Full-screen formats are a different story: they always use `AdManagerInterstitialAd` /
`GAMInterstitialAd` and `AdManagerAdRequest`, so interstitial/rewarded/app-open Direct
Access **may already work today** — untested, and worth exactly one experiment if anyone
still cares.

**The fix for banners is a one-line fork** — Apache-2.0, so this is legally free:

```java
public static boolean isAdManagerUnit(String unitId) {
  if (unitId == null) return false;
  return unitId.startsWith("/") || unitId.startsWith("ca-mb-app-pub-");
}
```

Plus the iOS equivalent. That is the entire change. It is a fork, not a config change —
but a two-line one, on a permissive licence.

**However**, per `ADMOB_ADX_HANDOFF.md` §5, Direct Access still doesn't fit our business
model (no per-publisher attribution → cannot pay publishers correctly). So the fork is
technically trivial and strategically pointless. Don't build it.

---

## What this means for our GAM/AdX setup — the actual good news

Our existing MCM ad units (`/22846411849,23205397816/JBM_AVD_VN_banner`) start with `/`.

**They will work on this library, unmodified, today.** No fork. Every format, both platforms.

And `customTargeting` ([RequestOptions.ts:119](../src/types/RequestOptions.ts#L119)) means
we can pass per-publisher key-values on the ad request, letting line items target and
report on a publisher dimension **without** minting a unit per publisher. That is the
onboarding-friction problem the whole investigation started from — solvable on a fully
supported path, combined with MCM Manage Inventory from handoff §6.

---

## Corrections to `ADMOB_ADX_HANDOFF.md`

| Line | Claim | Status |
|---|---|---|
| 123 | "Docs mention no Ad Manager / GAM / AdX support — AdMob only" | **wrong** — full GAM support in code, and [docs/displaying-ads.mdx:520](../docs/displaying-ads.mdx#L520) does document `GAMBannerAd` |
| 125-128 | "Separate GAM-specific RN packages exist, which hints the popular one does not cover GAM" | **bad inference** — those packages predate/duplicate this one's GAM support |
| 133-139 | "That single line of Java decides whether this is a config change or a fork" | **answered** — it's `AdManagerAdView`, conditionally. GAM = config change. Direct Access banners = 2-line fork. |
| 121 | "It cannot create ad units. It only forwards the unitId you pass." | **confirmed correct** |

---

## Remaining unknowns

- Whether full-screen Direct Access (`ca-mb-app-pub-XXXX/`) actually loads through the
  always-AdManager path. Untested; would need a real property code and a device.
- Whether MCM MI child-code reporting granularity is sufficient — unchanged from the
  handoff. That is a GAM console question, not a library question.

---

## Superseded / extended by

This document answered the AdView-vs-AdManagerAdView question. The negative findings
(no ads.txt, no network, no auto-generation) were later proven exhaustively rather than
asserted — see **PROOF_NO_AUTO_ADUNITS.md** in this folder, which contains the
reproducible commands, the full 153-file / 13,903-line audit scope, and the adversarial
falsification attempts (config-file mechanism, generic preferences API, reflection,
obfuscation vectors). Cite that file, not this one, when a finding is challenged.
