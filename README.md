# RN Pre-Release Audit Agent

> **Catch every App Store & Play Store rejection reason before you submit — a Claude AI skill for React Native developers**

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/bharmalhusen/rn-prerelease-audit)
[![Claude Skill](https://img.shields.io/badge/Claude-Skill-blueviolet.svg)](https://skillsmp.com)
[![React Native](https://img.shields.io/badge/React%20Native-0.70%2B-61dafb.svg)](https://reactnative.dev)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**Built by [Bharmal Husen](https://bharmalhusen.github.io) · React Native Engineer**  
[Portfolio](https://bharmalhusen.github.io) · [GitHub](https://github.com/bharmalhusen) · [LinkedIn](https://www.linkedin.com/in/husen-bharamal-601099113)

---

A Claude Code skill that performs a structured, deep pre-release audit of any React Native app before App Store or Google Play Store submission. It reads your project files, runs 30+ compliance checks, and outputs a prioritised report of every issue that can get your app **rejected, removed, or flagged** — in seconds.

No more last-minute rejections. No more store policy rules you didn't know existed.

---

## Why This Exists

Every React Native developer has been there: you submit a build after weeks of work and Apple or Google bounces it for a rule you could have caught in five minutes. This skill codifies every common rejection reason — permissions, billing compliance, ATT, Privacy Manifests, Target SDK, ProGuard, Hermes, and more — into a single repeatable checklist you run before every release.

---

## What It Checks — 30+ Checks Across 20 Categories

| # | Category | What It Catches |
|---|---|---|
| 1 | 📡 **App Tracking Transparency** | Missing `NSUserTrackingUsageDescription`, no `requestTrackingAuthorization` call, late ATT prompt |
| 2 | 🍎 **Sign in with Apple** | Third-party login without Apple login (guideline 4.8), missing entitlement |
| 3 | 🔏 **Auth Session Handling** | Tokens not cleared on logout, missing token refresh, no re-auth on destructive actions |
| 4 | 💳 **Payments & Billing** | No IAP for digital goods (iOS 3.1.1), external payment links on iOS, missing Play Billing |
| 5 | 🔐 **Permissions Audit** | Declared but unused permissions, used but undeclared, missing runtime request handling |
| 6 | 🖼 **Icons & Assets** | Missing densities, alpha channel on iOS icons (hard rejection), broken asset paths |
| 7 | 🌅 **Splash Screen** | Missing splash config, broken image paths, missing `resizeMode` |
| 8 | 🔒 **iOS Privacy Descriptions** | Missing `NS*UsageDescription` keys (hard rejection), descriptions under 10 chars |
| 9 | 🌐 **Network Security** | `usesCleartextTraffic=true`, `NSAllowsArbitraryLoads=true` |
| 10 | 🔑 **Secrets & API Keys** | OpenAI, Google, Firebase, GitHub tokens hardcoded in config files; tokens in AsyncStorage |
| 11 | 📊 **Crash Monitoring** | No Sentry/Crashlytics/Datadog installed or installed but never initialised |
| 12 | 🏗 **New Architecture** | Inconsistent `newArchEnabled` flags across Gradle and app.json |
| 13 | 🔁 **Network Resilience** | No timeout config, no retry logic, no offline/NetInfo handling |
| 14 | 🔗 **Deep Links** | App Links missing `autoVerify`, deep link config on one platform only |
| 15 | 📦 **Deprecated Dependencies** | `react-native-camera`, old async-storage package, unpinned `"*"` versions |
| 16 | 🗑️ **Account Deletion** | Apps with sign-up but no delete flow (Play Store policy — removal risk) |
| 17 | 🔒 **Privacy Policy** | No in-app privacy policy link (both stores require this) |
| 18 | 📂 **Target SDK** | `targetSdkVersion` below Play Store floor (≥ 34) — blocks all uploads |
| 19 | 👵 **Back-Press Handling** | No `BackHandler` on root navigator — app exits immediately on Back press |
| 20 | 📝 **Privacy Manifests** | Missing `PrivacyInfo.xcprivacy`, empty `NSPrivacyAccessedAPITypes` (Apple rejects on upload) |
| 21 | 🔗 **ProGuard / R8** | `minifyEnabled false` in release build, missing `proguard-rules.pro` |
| 22 | 🔐 **Export Compliance** | Missing `ITSAppUsesNonExemptEncryption` in Info.plist (manual click required every build) |
| 23 | ⚡ **Hermes Engine** | Hermes disabled on Android or iOS, inconsistent engine between platforms |
| 24 | 📦 **Bundle Size** | Assets > 5 MB, total assets > 30 MB, unused assets inflating download size |
| 25 | 🎨 **UI Safety** | No `ErrorBoundary`, missing loading states on data-fetching screens |
| 26 | ⚙️ **Android Activity Config** | Missing `android:exported`, wrong `windowSoftInputMode`, backup rules |
| 27 | 🎯 **Version Consistency** | `versionCode` / `versionName` / `CFBundleVersion` out of sync across files |
| 28 | 📱 **Screen Orientation** | Orientation not set or mismatched between app.json and native files |

---

## Sample Report Output

```
## RN Audit Report — MyApp

8 errors · 5 warnings · 3 info · 14 passed

### 🔴 Errors (must fix before release)

[Target SDK] — targetSdkVersion 33 is below Play Store minimum of 34
Fix: android/app/build.gradle → change `targetSdkVersion 33` to `targetSdkVersion 34`

[Privacy Manifest] — ios/PrivacyInfo.xcprivacy is missing
Fix: Create ios/PrivacyInfo.xcprivacy with NSPrivacyAccessedAPITypes entries for UserDefaults and FileTimestamp

[ATT] — NSUserTrackingUsageDescription missing from Info.plist while Firebase Analytics is present
Fix: Add NSUserTrackingUsageDescription key to ios/MyApp/Info.plist with a user-facing explanation

### 📋 Next Steps
1. Fix targetSdkVersion (android/app/build.gradle:12) — blocks upload entirely
2. Add PrivacyInfo.xcprivacy (ios/) — Apple rejects on Transporter upload before review
3. Add ITSAppUsesNonExemptEncryption to Info.plist — eliminates manual step on every build
```

---

## How to Use

### Option 1 — Claude Code skill (recommended)

```bash
# Inside any React Native project
/rn-prerelease-audit
```

### Option 2 — Claude.ai

1. Copy the full contents of [`audit.md`](./audit.md)
2. Paste as the **system prompt** when starting a new Claude conversation
3. Type: `Audit the React Native project at /path/to/your/app`

### Option 3 — Claude Projects

1. Create a new Project at [claude.ai](https://claude.ai)
2. Paste `audit.md` as the Project Instructions
3. Start a conversation and point it at your project directory

---

## Common Rejections This Prevents

| Store | Rejection Reason | Check |
|---|---|---|
| App Store | ATT prompt missing while analytics SDK present | 📡 ATT |
| App Store | No Sign in with Apple alongside Google/Facebook login | 🍎 Apple Login |
| App Store | Icon has alpha channel | 🖼 Icons |
| App Store | `NSUsageDescription` key missing | 🔒 Privacy Descriptions |
| App Store | `PrivacyInfo.xcprivacy` absent | 📝 Privacy Manifests |
| App Store | External payment link in app | 💳 Billing |
| Play Store | `targetSdkVersion` below 34 | 📂 Target SDK |
| Play Store | No account deletion option for apps with sign-up | 🗑️ Account Deletion |
| Play Store | Digital goods sold without Google Play Billing | 💳 Billing |

---

## About

**Bharmal Husen** — React Native Engineer focused on mobile product quality, App Store compliance, and AI tooling.

| | |
|---|---|
| 🌐 Portfolio | [bharmalhusen.github.io](https://bharmalhusen.github.io) |
| 💼 LinkedIn | [husen-bharamal-601099113](https://www.linkedin.com/in/husen-bharamal-601099113) |
| 🐙 GitHub | [github.com/bharmalhusen](https://github.com/bharmalhusen) |
| 📧 Email | bharamalhusen@gmail.com |

---

## Contributing

Found a new rejection reason not covered here? Open a PR:

1. Add the check to `audit.md` following the existing pattern
2. Add a row to the checks table in this README
3. Include the severity (error / warning / info) and the exact store policy it maps to

---

## License

MIT — free to use, fork, and share. A GitHub star helps other React Native developers find this.
