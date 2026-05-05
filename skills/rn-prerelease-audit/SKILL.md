---
name: rn-prerelease-audit
description: Deep pre-release audit for React Native apps covering 25+ App Store and Play Store compliance checks — permissions, secrets, Hermes, ProGuard, PrivacyInfo.xcprivacy, ATT, back-press handling, export compliance, and more.
author: Bharmal Husen
tags: [react-native, ios, android, app-store, play-store, audit, compliance, expo, hermes, proguard, permissions]
version: 1.0.0
---

# RN Pre-Release Audit Agent

You are a React Native pre-release audit expert. Perform a deep audit and produce a structured report.

## Step 1 — Read project files

Read (skip if not found):
- `android/app/src/main/AndroidManifest.xml`
- `android/app/build.gradle`
- `android/gradle.properties`
- `ios/**/Info.plist`
- `app.json`, `app.config.js` / `app.config.ts`
- `package.json`
- `ios/Podfile`

Scan all `.ts`, `.tsx`, `.js`, `.jsx` (excluding node_modules, .expo, android, ios) for:
- **Permission keywords**: `PermissionsAndroid`, `requestPermission`, `CAMERA`, `RECORD_AUDIO`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `Geolocation`, `ImagePicker`, `Camera`, `Audio`, `Contacts`, `MediaLibrary`, `Notifications`, `Bluetooth`
- **Analytics/tracking keywords**: `firebase`, `analytics`, `amplitude`, `mixpanel`, `segment`, `facebook`, `appsflyer`, `branch`, `adjust`, `crashlytics`, `sentry`, `datadog`, `axios`, `fetch`, `XMLHttpRequest`

## Step 2 — Run all checks

### 📡 ATT — iOS
If analytics/network keywords found in Step 1:
- `NSUserTrackingUsageDescription` missing or < 20 chars in Info.plist → **error**
- `requestTrackingAuthorization` not found in source → **error**
- `requestTrackingAuthorization` not called before analytics SDK init in root component → **warning**

If no analytics found → **info**

### 🍎 Sign in with Apple
Search for third-party login: `GoogleSignin`, `google-signin`, `LoginWithGoogle`, `LoginButton`, `LoginManager`, `signInWithFacebook`, `TwitterSignIn`, `GithubAuthProvider`, `OAuthProvider`

If found:
- No Sign in with Apple (`appleAuth`, `apple-authentication`, `AppleButton`, `expo-apple-authentication`, `@invertase/react-native-apple-authentication`) → **error**
- Apple sign-in button not at same visual prominence as other social buttons → **warning**
- `com.apple.developer.applesignin` missing from entitlements or `app.json` → **error**

If no third-party login → **info**

### 🔏 Auth session handling
Search for: `login`, `signIn`, `token`, `accessToken`, `refreshToken`, `session`, `jwt`, `bearer`

If found:
- **AUTH-001**: logout handler (`logout`, `signOut`, `clearSession`) doesn't clear token + storage (`AsyncStorage.clear`, `removeItem`, `SecureStore.deleteItemAsync`, `Keychain.resetGenericPassword`) → **warning**
- **AUTH-002**: auth tokens used but no refresh/expiry logic (`refreshToken`, `401`, `intercept`, `onTokenExpired`) → **warning**
- **AUTH-003**: sensitive actions (`deleteAccount`, `changePassword`, `transferFunds`) without re-auth (`LocalAuthentication`, `confirmPassword`) → **warning**

### 💳 Payments & billing
Search for: `purchase`, `subscribe`, `subscription`, `checkout`, `iap`, `inAppPurchase`, `sku`, `offering`, `premium`, `upgrade`
Search separately for donations: `donate`, `donation`, `tip`, `charity`, `nonprofit`

If digital content signals found:
- **iOS**: no IAP library (`react-native-iap`, `expo-in-app-purchases`, `@revenuecat/purchases-react-native`, `StoreKit`) → **error**; external payment SDK (`stripe`, `razorpay`, `paypal`) or `openURL` near purchase flow → **error**
- **Android**: no Play Billing (`react-native-iap`, `@revenuecat/purchases-react-native`, `BillingClient`) → **error**; external payment links → **warning**

If only donation signals: iOS without IAP in a for-profit app → **error**; Android external processors → **info**

### 🔐 Permissions audit
- List all permissions in AndroidManifest.xml and Info.plist
- Declared but unused in code → **warning**; used in code but undeclared → **error**; empty/very short usage description → **error**

Runtime handling for each permission-gated feature:
- **Android**: no `PermissionsAndroid.request()` before feature → **error**; `granted`/`denied`/`never_ask_again` states not all handled → **warning**; no `check()` before `request()` → **warning**
- **iOS**: no `check()` before gated API → **warning**; `.blocked`/`.denied` doesn't open settings via `Linking.openURL('app-settings:')` → **warning**; `.limited` not handled separately → **info**
- **Both**: permission `request()` at app launch without user-triggered action → **warning**

### 🖼 Icons & assets
- Check `app.json`/`app.config` icon paths exist on disk
- Bare RN: `android/.../mipmap-*/ic_launcher.png` in all densities (mdpi–xxxhdpi) → missing → **error**
- iOS: `ios/*/Images.xcassets/AppIcon.appiconset/Contents.json` exists; run `sips -g hasAlpha <file>` on each listed image → `hasAlpha: yes` → **error**; non-.png extension → **error**

### 🌅 Splash screen
- `splash.image` or `expo-splash-screen` config present; file exists; `resizeMode` set → missing → **warning**
- Bare RN: `android/.../res/drawable/` launch resources exist

### 🔒 iOS privacy descriptions
For each permission in Info.plist, matching `NS*UsageDescription` key must exist and be ≥ 10 chars → missing → **error**; too short → **warning**

Check: `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSLocationAlwaysAndWhenInUseUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSContactsUsageDescription`, `NSBluetoothAlwaysUsageDescription`, `NSFaceIDUsageDescription`

### 🌐 Network security
- Android `android:usesCleartextTraffic=true` → **error**; build variable → **warning**; check for `network_security_config` reference
- iOS `NSAllowsArbitraryLoads=true` → **error**

### 🔑 Secrets & API keys
Scan `app.json`, `app.config.js`, `.env`, `.env.production`, `package.json` for:
- `sk-[a-zA-Z0-9]{20,}`, `AIza[a-zA-Z0-9_-]{35}`, `AAAA[a-zA-Z0-9_-]{140,}`, `gh[pousr]_[a-zA-Z0-9]{36}`, `EAA[a-zA-Z0-9]{50,}`, any key named `SECRET`/`API_KEY`/`TOKEN`/`PASSWORD` with hardcoded value → **error**

**SEC-003 — Secure token storage**
- `AsyncStorage.setItem` near `token`/`accessToken`/`jwt` → **error**
- Auth tokens found but no `react-native-keychain`/`expo-secure-store` in package.json → **error**

### 📊 Crash monitoring
- No crash library (`@react-native-firebase/crashlytics`, `@sentry/react-native`, `@datadog/mobile-react-native`, `bugsnag-react-native`) → **warning**
- Library found but init call (`Sentry.init`, `Crashlytics().setCrashlyticsCollectionEnabled`, `bugsnag.start`) missing in source → **warning**
- Init not in entry file (`index.js`, `App.tsx`) → **info**

### 🏗 New Architecture
- `newArchEnabled=true` in `android/gradle.properties`
- `hermesEnabled` in `android/app/build.gradle`
- `expo-build-properties` `newArchEnabled` in `app.json`
- RN ≥ 0.76 has New Arch on by default
- Inconsistent flags across files → **warning**

### 🔁 Network resilience
Search for API calls: `fetch`, `axios`, `useQuery`, `useMutation`, `api.get`, `api.post`

- **NET-001**: no timeout config (`timeout:`, `AbortController`, `signal`) → **warning**
- **NET-002**: no retry logic (`retry`, `axios-retry`, `retryDelay`) → **info**
- **NET-003**: no `NetInfo`/`useNetInfo`/`isConnected` usage → **warning**; NetInfo used but no visible offline fallback UI → **warning**

### 🔗 Deep links
- AndroidManifest `intent-filter` with `VIEW` + custom scheme; Info.plist `CFBundleURLTypes`
- One platform configured, other not → **warning**
- `android:autoVerify="true"` missing for HTTPS links → **warning**

### 📦 Dependencies
Flag in package.json:
- `react-native-camera` → deprecated → **warning**
- `@react-native-community/async-storage` → moved → **warning**
- `react-native-splash-screen` → deprecated → **warning**
- `react-native-firebase` (non-scoped) → **warning**
- RN < 0.73 → **warning**
- Any package with `"*"` version → **error**

### 🗑️ Account deletion (Android)
Search for sign-up signals: `signUp`, `register`, `createAccount`, `createUser`, `SignUpScreen`, `RegisterScreen`

If found:
- No deletion signals (`deleteAccount`, `deleteUser`, `removeAccount`, `DeleteAccountScreen`) → **error**
- Deletion not reachable from settings/profile screen → **warning**
- Deletion doesn't also delete associated data → **warning**

If no sign-up signals → **info**

### 🔒 Privacy policy
- No `privacyPolicy`/`privacy-policy`/`Privacy Policy`/`terms`/`termsOfService` in source → **error**
- Not reachable from settings, profile, or onboarding screen → **warning**
- No `privacyPolicyUrl` in `app.json` → **warning**

### 📦 Assets & performance
- **PERF-001**: `find . -not -path "*/node_modules/*" -size +1M` → single asset > 5 MB → **warning**; total assets > 30 MB → **warning**
- **PERF-002**: each file in `assets/` not referenced in source → **warning**

### 🎨 UI safety
- **UI-002**: no `ErrorBoundary`/`componentDidCatch`/`react-error-boundary` wrapping root navigator → **warning**
- **UI-003**: data fetching (`useQuery`, `useSWR`, `useEffect`+`fetch`) without `ActivityIndicator`/`Skeleton`/`isLoading` nearby → **warning**

### ⚙️ Android activity config
- `android:exported="true"` on MainActivity → missing → **error**
- `android:windowSoftInputMode="adjustResize"` → **warning**
- `android:allowBackup="false"` missing → **warning**
- `android:dataExtractionRules` missing → **warning**
- `SYSTEM_ALERT_WINDOW` permission present → **warning**
- `WRITE_EXTERNAL_STORAGE` present → **error**

### 🎯 Version consistency
- `versionCode`/`versionName` in `build.gradle`; `CFBundleVersion`/`CFBundleShortVersionString` in Info.plist; `version` in `app.json`/`package.json`
- Inconsistent across files → **warning**; versionCode not integer or versionName not semver → **warning**

### 📱 Orientation
- `android:screenOrientation` or `orientation` in `app.json` missing → **info**; mismatched across files → **warning**

### 📂 Android Target SDK
- `targetSdkVersion` < 34 → **error**; variable reference that can't be resolved → **warning**
- `compileSdkVersion` < `targetSdkVersion` → **error**; either value missing → **error**

### 👵 Android back-press
Search for `BackHandler`, `useBackHandler`:
- None found and root stack navigator present → **warning**
- `BackHandler.addEventListener('hardwareBackPress', ...)` not on root/home screen → **warning**
- Handler returns `false`/`undefined` unconditionally → **warning**
- No navigator and no BackHandler → **info**

### 📝 PrivacyInfo.xcprivacy — iOS
- `ios/PrivacyInfo.xcprivacy` or `ios/<AppName>/PrivacyInfo.xcprivacy` missing → **error**
- If exists: `NSPrivacyTracking=false` while analytics SDKs present → **warning**; `NSPrivacyTrackingDomains` empty while analytics present → **warning**; `NSPrivacyAccessedAPITypes` empty → **warning**
- Fingerprinting signals (`UIDevice.identifierForVendor`, `sysctlbyname`, `mach_absolute_time`) without matching `NSPrivacyAccessedAPITypes` entry → **error**

### 🔗 ProGuard / R8
In `android/app/build.gradle` release block:
- `minifyEnabled false` or missing → **warning**
- `minifyEnabled true` but no `proguardFiles` entry → **warning**
- `shrinkResources false` or missing → **info**
- `minifyEnabled true` + `proguardFiles` present → **passed**

### 🔐 Export compliance — iOS
In `ios/**/Info.plist`:
- `ITSAppUsesNonExemptEncryption` key missing → **warning**
- `= false` → **passed**
- `= true` but `ITSEncryptionExportComplianceCode` missing → **warning**
- Custom crypto signals (`CryptoKit`, `CommonCrypto`, `crypto-js`, `aes`, `rsa`, `pbkdf2`) found while key is `false` → **warning**

### ⚡ Hermes
**Android** (`android/app/build.gradle`, `android/gradle.properties`):
- RN < 0.70: `enableHermes: true` missing → **error**
- RN ≥ 0.70: explicit `enableHermes: false` or `hermesEnabled=false` → **error**

**iOS** (`ios/Podfile`):
- RN < 0.70: `:hermes_enabled => true` missing → **error**
- RN ≥ 0.70: `:hermes_enabled => false` → **error**
- Expo: `expo-build-properties` `"useHermes": false` → **error**
- Hermes enabled on one platform but not the other → **warning**

## Step 3 — Output the report

---

## RN Audit Report — [app name from package.json]

**[X errors · Y warnings · Z info · W passed]**

---

### 🔴 Errors (must fix before release)

> **[Category]** — [issue with exact file/key/value]
>
> Fix: `exact fix`

---

### 🟡 Warnings (should fix)

> **[Category]** — [issue]
>
> Fix: `exact fix`

---

### 🔵 Info

> **[Category]** — [observation]

---

### ✅ Passed

One line per passing check.

---

### 📋 Next steps

Top 3 fixes in priority order with exact file and line to change.
