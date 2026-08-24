# AdMob / AdX Ad-Unit Research — Handoff

**Date:** 24 Aug 2026
**Continue this in:** a clone of `react-native-google-mobile-ads`

---

## The original question

Manager observed a publisher whose app used AdMob but who "did not need to create any ad unit ID" — ad units either weren't needed or were created automatically. Goal: find out how, and whether we can do the same for our Google AdX / Google Ad Manager setup.

**Manager has since confirmed** the publisher used this package:
https://www.npmjs.com/package/react-native-google-mobile-ads

---

## Findings so far (verified against live docs)

### 1. AdMob has NO built-in ad-unit-free mode — CONFIRMED

The ad unit ID is a required parameter on every ad request. The SDK cannot build a request without it. There is no mode that takes only an App ID and generates a unit at runtime.

- https://support.google.com/admob/answer/7356431?hl=en
- https://support.google.com/admob/answer/7311346?hl=en

### 2. Automated creation is possible, but YOU build it

Google exposes `accounts.adUnits.create` — an API endpoint, **not** a Google-run feature. A backend can create units via API, store the IDs, and serve them to the app at startup. The developer never opens the AdMob console and would honestly say "I never created an ad unit" — but one exists.

**Note: limited access — requires approval from Google.**

- https://developers.google.com/admob/api/reference/rest/v1beta/accounts.adUnits/create?hl=en

### 3. AdX Direct Access — a real no-ad-unit path (AdX only, not AdMob)

It is a change in app code, not a GAM setting:

```kotlin
// Normal GAM
adView.adUnitId = "/22846411849/JBM_AVD_VN_banner"

// Direct Access
adView.adUnitId = "ca-mb-app-pub-5629679302779023/"   // trailing slash MANDATORY
```

- Uses `AdManagerAdView` + `AdManagerAdRequest` (NOT `AdView`)
- Ad Manager App ID still required in the manifest
- Supports banner, interstitial, native, rewarded — Android and iOS
- Property code comes from **Admin → Linked accounts** (already exists in our account)
- There is NO "Direct Access" tab in GAM. Nothing to enable.
- Missing slash error: `Invalid Request. Cannot determine request type. Is your ad unit id correct?`

Docs:
- https://developers.google.com/ad-manager/mobile-ads-sdk/android/next-gen/adx-direct?hl=en
- https://developers.google.com/ad-manager/mobile-ads-sdk/ios/adx-direct
- https://support.google.com/admanager/answer/188529?hl=en

### 4. Longevity risk — REAL BUT OVERSTATED EARLIER

Trial exhibit **PTX0945**, "AdX Direct Deprecation Plan", from *US v. Google* (ad tech case). Court testimony described AdX Direct as **under 2% of AdX revenue**, "not effective", "not widely used".

**Caveats — state these honestly:**
- It is a PUBLIC court exhibit, not an "internal doc we found"
- Undated; largely concerns **web** AdX Direct tags, not confirmed to cover mobile-app Direct Access
- Google's mobile docs are live, maintained, and ported to the next-gen SDK
- No sunset appears on Google's published deprecation schedule
- Direct justice.gov fetch returned 403 — relying on secondary reporting

Links:
- https://ppc.land/google-reveals-internal-plans-for-adx-shutdown-during-federal-antitrust-trial/
- https://justice.gov/atr/media/1412561/dl?inline=
- https://developers.google.com/ad-manager/mobile-ads-sdk/android/deprecation

### 5. Direct Access does not fit our business model

Removing the ad unit removes the object all per-placement control hangs off.

**Kept (property level):** geo + pricing via an AdX line item with the web property alias, brand safety, blocking controls.

**Lost (per placement):**
- per-placement floor prices
- per-placement frequency caps and refresh rate
- per-placement reporting
- **separation of one publisher's traffic from another's** ← disqualifying

Our inventory already encodes per-publisher floors in naming (`..._$0.40`) and per-placement splits (`V1`–`V5`). Under Direct Access every publisher firing the same property code becomes one undifferentiated blob — we could not attribute revenue, so we could not pay publishers correctly.

Direct Access is built for a single publisher monetizing their own app. We are a network monetizing many. Ad units are precisely what separates them.

### 6. We may already have the answer: MCM Manage Inventory

Our ad units look like `/22846411849,23205397816/JBM_AVD_VN_banner` — parent network + child network code. That is MCM Manage Inventory. Ad units live in OUR network; one parent ad unit can serve many child publishers, with the child code separating reporting. Reduces per-publisher ad unit creation WITHOUT losing controls or attribution.

- https://support.google.com/admanager/answer/9611105?hl=en
- https://support.google.com/admanager/answer/9579170?hl=en

---

## GAM concepts (clarified during research)

- **Ad unit** = a labelled slot. NOT where control lives. No geo, no pricing on it. (Confirmed against our own ad unit screen: name, sizes, reward, frequency caps, refresh rate, placements, labels — that is all.)
- **Line item** = "a deal to show ads." Geo, CPM, budget, dates, priority, device, creative. THIS is where control lives.
- **Creative** = the actual image/video. Line item = the rules.
- **Yield group** = lets multiple networks (AdX, Meta, Unity) compete for one slot. Mediation.

---

## About the package

`react-native-google-mobile-ads` — third-party **wrapper/translator**, by Invertase (not Google). JS cannot call the native Kotlin/Swift SDK directly, so this bridges it.

```
React Native JS → react-native-google-mobile-ads → native GMA SDK → Google ad servers
```

- v16.5.0, actively maintained, 638 files, ~1.29 MB unpacked
- **License: Apache-2.0** — free to fork, modify, ship commercially
- Repo is public: https://github.com/invertase/react-native-google-mobile-ads
- Has SLSA provenance attestation
- Google has official SDKs for Android/iOS/Unity/Flutter — NOT React Native. Hence third party.
- **It cannot create ad units.** It only forwards the `unitId` you pass. No credentials on device.

Docs mention **no** Ad Manager / GAM / AdX support — AdMob only. BUT that is from the docs page only; **docs often lag code. Check the source.**

Separate GAM-specific RN packages exist, which itself hints the popular one does not cover GAM:
- https://github.com/simpleTechs/react-native-ad-manager
- https://www.npmjs.com/package/@callosum/react-native-google-ad-manager

---

## THE KEY QUESTION FOR THE NEXT SESSION

**Does the Android native bridge instantiate `AdView` or `AdManagerAdView`?**

- `AdView` (AdMob) → Direct Access + GAM paths impossible without modifying the library
- `AdManagerAdView` → GAM paths may already work undocumented, and `ca-mb-app-pub-XXXX/` might work today

That single line of Java decides whether this is a config change or a fork.

Where to look:

```bash
grep -rn "AdManagerAdView\|AdManagerAdRequest" android/src/
grep -rn "new AdView\|AdView(" android/src/
grep -rn "GAMBannerView\|GAMRequest" ios/          # iOS equivalents
```

Also: `RNGoogleMobileAdsExample/` is a full working sample app in the repo — likely faster to test with than building from scratch.

---

## Open questions to confirm with manager

1. Did the observed publisher have their own AdMob account, or were they onboarded through a platform? (If a platform: backend API provisioning is essentially certain.)
2. Does the `$0.40` naming in our inventory really mean per-publisher floors? (Inferred, not confirmed.)
3. Ask our Google rep whether AdX Direct Access is still provisioned for new mobile properties.

---

## Recommended next steps

1. Read the native bridge source — answer the `AdView` vs `AdManagerAdView` question.
2. Test whether MCM MI gives the per-child reporting granularity we need (likely solves the real onboarding problem on a supported path).
3. Optionally POC Direct Access in **Kotlin** (not React Native) to measure exactly what reporting is lost. Kotlin because that is where Google documents it — testing the mechanism, not someone's wrapper.
4. Ask the Google rep about Direct Access's future before any build.

---

## Paste this to resume in the cloned repo

> Researching whether `react-native-google-mobile-ads` supports Google Ad Manager / AdX Direct Access (`ca-mb-app-pub-XXXX/` property codes) or is AdMob-only. Manager confirmed this is the package a publisher used who reportedly never created ad unit IDs. Need to know: does the native bridge use `AdView` or `AdManagerAdView`, and can a GAM path be passed as `unitId`?
