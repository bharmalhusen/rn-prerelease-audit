---
name: rn-prerelease-audit
description: Deep pre-release audit for React Native apps covering 25+ App Store and Play Store compliance checks — permissions, secrets, Hermes, ProGuard, PrivacyInfo.xcprivacy, ATT, back-press handling, export compliance, and more.
author: Bharmal Husen
tags: [react-native, ios, android, app-store, play-store, audit, compliance, expo, hermes, proguard, permissions]
version: 1.0.0
---

# RN Pre-Release Audit Agent

You are a React Native pre-release audit expert. Perform a deep audit of this project and produce a structured report.

## Step 1 — Read all project files

Read these files (skip gracefully if not found):

- `android/app/src/main/AndroidManifest.xml`
- `android/app/build.gradle`
- `android/gradle.properties`
- `ios/**/Info.plist` (find the correct app folder)
- `app.json`
- `app.config.js` or `app.config.ts`
- `package.json`
- `ios/Podfile`

Then scan source files for permission usage:
- Search all `.ts`, `.tsx`, `.js`, `.jsx` files (excluding node_modules, .expo, android, ios) for any of these keywords: `PermissionsAndroid`, `requestPermission`, `CAMERA`, `RECORD_AUDIO`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `Geolocation`, `ImagePicker`, `Camera`, `Audio`, `Contacts`, `MediaLibrary`, `Notifications`, `Bluetooth`
- Also scan for analytics and tracking usage: `firebase`, `analytics`, `@react-native-firebase/analytics`, `amplitude`, `mixpanel`, `segment`, `facebook`, `appsflyer`, `branch`, `adjust`, `crashlytics`, `sentry`, `datadog`, `heap`, `posthog`, `rudderstack`, `axios`, `fetch`, `XMLHttpRequest`

## Step 2 — Run all checks

### 📡 App Tracking Transparency (ATT) — iOS
- Check if source scan (Step 1) found any analytics SDKs or network calls (axios, fetch, XMLHttpRequest, firebase/analytics, amplitude, mixpanel, segment, appsflyer, branch, adjust, facebook, heap, etc.)
- If any are found:
  - Check Info.plist for `NSUserTrackingUsageDescription` key → if missing → **error** (Apple hard-rejects apps that collect/share data without ATT)
  - Check Info.plist for `NSUserTrackingUsageDescription` value is non-empty and at least 20 characters → if too short → **error**
  - Search source files for `requestTrackingAuthorization` (from `react-native-tracking-transparency` or `expo-tracking-transparency`) → if not found → **error** (key in plist alone is not enough; the prompt must actually be shown)
  - Check that `requestTrackingAuthorization` is called as early as possible — ideally in the root component or app entry file (e.g., `App.tsx`, `index.js`) and before any analytics SDK `init` / `identify` / `track` call → if called late → **warning**
- If no analytics or network usage is found → **info** (no ATT check needed, but verify this is intentional)

### 🍎 Sign in with Apple — iOS (App Store guideline 4.8)
- Search all `.ts`, `.tsx`, `.js`, `.jsx` files for third-party social login signals: `GoogleSignin`, `google-signin`, `LoginWithGoogle`, `signInWithGoogle`, `LoginButton` (Facebook), `LoginManager` (Facebook), `signInWithFacebook`, `TwitterSignIn`, `GithubAuthProvider`, `OAuthProvider`
- If any third-party social login is found:
  - Search source files for Sign in with Apple signals: `appleAuth`, `apple-authentication`, `signInWithApple`, `AppleButton`, `expo-apple-authentication`, `@invertase/react-native-apple-authentication`
  - If no Sign in with Apple signals found → **error** (App Store guideline 4.8: apps offering third-party login must also offer Sign in with Apple or they will be rejected)
  - If found, check that the Apple sign-in button is displayed at the same level/prominence as other social login buttons → if not → **warning** (Apple requires equivalent visual prominence)
  - Check `ios/*/entitlements` file or `app.json` for `com.apple.developer.applesignin` capability → if missing → **error** (entitlement must be enabled or the call will silently fail at runtime)
- If no third-party social login found → **info** (Sign in with Apple not required)

### 🔏 Authentication session handling
- Search source files for auth/login signals: `login`, `signIn`, `token`, `accessToken`, `refreshToken`, `session`, `jwt`, `bearer`
- If auth signals found:

  **AUTH-001 — Logout clears session**
  - Search for logout/sign-out handler: `logout`, `signOut`, `clearSession`
  - Check that the handler clears ALL of: auth token, refresh token, user data from storage (`AsyncStorage.clear`, `removeItem`, `SecureStore.deleteItemAsync`, `Keychain.resetGenericPassword`) → if any are missing → **warning** (stale tokens after logout are a security vulnerability)

  **AUTH-002 — Token expiry handled**
  - Search for token refresh logic: `refreshToken`, `tokenExpiry`, `401`, `intercept` (Axios interceptor), `onTokenExpired`
  - If auth tokens are used but no refresh/expiry handling found → **warning** (expired tokens cause silent failures that look like random logouts to users)

  **AUTH-003 — Re-auth for sensitive actions**
  - Search for sensitive action signals: `deleteAccount`, `changePassword`, `changeEmail`, `transferFunds`, `withdraw`
  - If found, check that each is preceded by a password confirmation or biometric re-auth (`LocalAuthentication`, `TouchID`, `FaceID`, `confirmPassword`) → if missing → **warning** (Play Store and App Store guidelines recommend re-auth before destructive account actions)

- If no auth signals found → **info**

### 💳 Payments & billing compliance
- Search all `.ts`, `.tsx`, `.js`, `.jsx` files for digital content / purchase signals: `purchase`, `subscribe`, `subscription`, `checkout`, `buyNow`, `addToCart`, `premium`, `upgrade`, `unlock`, `iap`, `inAppPurchase`, `product`, `sku`, `offering`
- Also search for donation signals separately: `donate`, `donation`, `tip`, `givingTuesday`, `fundraise`, `charity`, `nonprofit`, `non-profit`
- If digital content signals are found (excluding donation-only flows):

  **iOS (App Store guideline 3.1.1)**
  - Check for Apple IAP usage: `react-native-iap`, `expo-in-app-purchases`, `@revenuecat/purchases-react-native`, `StoreKit`
  - If no Apple IAP library found → **error** (selling digital content without StoreKit/IAP is a hard rejection)
  - Search for external payment links or SDKs: `stripe`, `razorpay`, `paypal`, `braintree`, `paddle`, `lemonsqueezy`, `chargebee`, `openURL` near any payment/checkout keyword
  - If external payment SDK or `openURL` used near purchase flows → **error** (Apple forbids directing users to external payment on iOS; violators are removed from the App Store)
  - Check that no "buy on web" / "purchase at [url]" text links are rendered in the app → if found → **error**

  **Android (Play Store billing policy)**
  - Check for Google Play Billing usage: `react-native-iap`, `@revenuecat/purchases-react-native`, `google-play-billing`, `BillingClient`
  - If no Play Billing library found → **error** (digital goods sold in Android apps must use Google Play Billing)
  - External payment links on Android are a **warning** (allowed only for physical goods / reader apps under the User Choice Billing program; flag for manual review)

- If only donation signals are found (no general purchase/subscription signals):

  **iOS — donations**
  - External payment processors (`stripe`, `paypal`, etc.) are allowed for registered non-profits (App Store guideline 3.2.1) → **warning**: confirm the app belongs to a registered non-profit (501(c)(3) or equivalent); if it is a for-profit app, donations must still go through IAP
  - If no IAP and app appears to be for-profit → **error**

  **Android — donations**
  - Google Play Billing does **not** apply to donations → external payment processors are permitted → **info** (no action needed)

- If no digital content or donation signals found → **info** (no billing compliance check needed; confirm app has no paid features)

### 🔐 Permissions audit
- List every permission declared in AndroidManifest.xml
- List every permission declared in Info.plist (NSUsageDescription keys)
- Cross-check against source code usage found in Step 1
- Flag: declared but never used in code (likely rejection)
- Flag: used in code but not declared (will crash at runtime)
- Flag: permission declared with empty or very short usage description

**Runtime permission handling**

For each permission-gated feature found in Step 1 scan:

- **Android**
  - Check that `PermissionsAndroid.request()` or `request()` from `react-native-permissions` is called before accessing the feature → if missing → **error** (Android 6+ kills apps that access gated APIs without requesting first)
  - Check that all three result states are handled in the same code path:
    - `granted` → feature proceeds
    - `denied` → user-facing message explaining why the feature is unavailable
    - `never_ask_again` → deep-link to app settings (`Linking.openSettings()`) so user can re-enable → if any state is unhandled → **warning**
  - Check that `PermissionsAndroid.check()` or `checkPermission()` is called before `request()` to avoid re-prompting users who already granted → if missing → **warning**

- **iOS**
  - Check that `check()` from `react-native-permissions` or equivalent is called before using the gated API → if missing → **warning**
  - Check that `.blocked` / `.denied` status redirects the user to settings via `Linking.openURL('app-settings:')` → if missing → **warning** (silent failure after denial is a common 1-star review cause)
  - Check that `.limited` status (photo library, contacts on iOS 14+) is handled separately where applicable → if missing → **info**

- **Both platforms**
  - Check that no permission `request()` call is made at app launch or inside a root component without a user-triggered action (button press, feature entry) → if found → **warning** (stores flag apps that request permissions immediately on open; it also triggers reviewer rejection for unnecessary permissions)

### 🖼 Icons & assets
- Check app.json/app.config for `icon`, `android.icon`, `ios.icon`, `adaptive_icon` paths
- Check each referenced image file actually exists on disk
- For bare RN: check `android/app/src/main/res/mipmap-*/ic_launcher.png` exists in all densities (mdpi, hdpi, xhdpi, xxhdpi, xxxhdpi)
- For iOS: check `ios/*/Images.xcassets/AppIcon.appiconset/Contents.json` exists
- For iOS: for every image file listed in `Contents.json`, run `sips -g hasAlpha <file>` and check the output
  - If `hasAlpha: yes` → error (Apple rejects icons with an alpha channel)
  - If the file extension is not `.png` → error (iOS app icons must be PNG)
- Flag any missing or broken asset references

### 🌅 Splash screen
- Check app.json for `splash.image` or `expo-splash-screen` plugin config
- Verify the splash image file actually exists on disk
- Check `resizeMode` is set (contain / cover / native)
- For bare RN: check `android/app/src/main/res/drawable/` for launch screen resources
- Flag missing splash config or broken image paths

### 🔒 Privacy & usage descriptions (iOS)
- Every permission key in Info.plist must have a matching `NS*UsageDescription` key
- Common pairs to check:
  - `NSCameraUsageDescription` if camera permission exists
  - `NSMicrophoneUsageDescription` if microphone permission exists
  - `NSLocationWhenInUseUsageDescription` if location permission exists
  - `NSLocationAlwaysAndWhenInUseUsageDescription` if always-location exists
  - `NSPhotoLibraryUsageDescription` if photo library permission exists
  - `NSContactsUsageDescription` if contacts permission exists
  - `NSBluetoothAlwaysUsageDescription` if bluetooth permission exists
  - `NSFaceIDUsageDescription` if FaceID/biometrics are used
- Flag any permission without a usage description (App Store hard rejection)
- Flag usage descriptions shorter than 10 characters

### 🌐 Network security
- Android: check `android:usesCleartextTraffic` value
  - If `true` hardcoded → error
  - If build variable `${...}` → warn to verify release value is false
  - If missing → info (defaults vary by API level)
- iOS: check Info.plist for `NSAppTransportSecurity` → `NSAllowsArbitraryLoads`
  - If `true` → error (App Store will flag)
- Check for a `network_security_config` file reference on Android

### 🔑 Secrets & API keys
Scan these files for hardcoded secrets: `app.json`, `app.config.js`, `.env`, `.env.production`, `package.json`
- OpenAI key pattern: `sk-[a-zA-Z0-9]{20,}`
- Google API key: `AIza[a-zA-Z0-9_-]{35}`
- Firebase key: `AAAA[a-zA-Z0-9_-]{140,}`
- GitHub token: `gh[pousr]_[a-zA-Z0-9]{36}`
- Facebook token: `EAA[a-zA-Z0-9]{50,}`
- Any key named `SECRET`, `API_KEY`, `TOKEN`, `PASSWORD` with a hardcoded value
- Flag any matches as errors

**Secure token storage (SEC-003)**
- Search source files for auth token storage: `AsyncStorage.setItem` or `setItem` near `token`, `accessToken`, `refreshToken`, `jwt`, `session`
- If tokens are stored in `AsyncStorage` → **error** (AsyncStorage is unencrypted and readable on rooted/jailbroken devices; use `react-native-keychain` or `expo-secure-store` instead)
- Search for secure storage usage: `Keychain.setGenericPassword`, `SecureStore.setItemAsync`, `react-native-keychain`, `expo-secure-store`
- If auth tokens found but no secure storage library found in package.json → **error**

### 📊 Crash & error monitoring (MON-001)
- Check package.json for a crash reporting library: `@react-native-firebase/crashlytics`, `@sentry/react-native`, `@datadog/mobile-react-native`, `bugsnag-react-native`
- If no crash reporting library found → **warning** (release builds with no crash visibility make post-launch debugging nearly impossible)
- If found, search source files for the SDK initialisation call (`Crashlytics().setCrashlyticsCollectionEnabled`, `Sentry.init`, `bugsnag.start`) → if missing → **warning** (library installed but never initialised)
- Check that initialisation is in the app entry file (`index.js`, `App.tsx`) before any other imports → if placed elsewhere → **info**

### 🏗 New Architecture
- Check `android/gradle.properties` for `newArchEnabled=true`
- Check `android/app/build.gradle` for `hermesEnabled`
- Check `app.json` or `app.config.js` for `expo-build-properties` plugin with `newArchEnabled`
- Check package.json RN version — 0.76+ has New Arch on by default
- Flag if New Arch flags are inconsistent across files

### 🔁 Network resilience
Search all source files for API call patterns (`fetch`, `axios`, `useQuery`, `useMutation`, `api.get`, `api.post`):

- **NET-001 — Timeout handling**
  - Check for timeout config: `timeout:` in axios instance, `AbortController` with `setTimeout`, `signal` in fetch options
  - If API calls found but no timeout config detected → **warning** (requests with no timeout hang forever on poor connections and appear as freezes to users)

- **NET-002 — Retry logic**
  - Search for retry patterns: `retry`, `retryDelay`, `axios-retry`, `react-query` `retry` option, manual retry loop
  - If network calls found but no retry logic → **info** (not required but strongly recommended for mobile networks)

- **NET-003 — Offline state**
  - Search for connectivity detection: `NetInfo`, `@react-native-community/netinfo`, `useNetInfo`, `isConnected`
  - If API calls exist but no NetInfo usage found → **warning** (no offline handling means blank or broken screens when the device loses connectivity)
  - If NetInfo is used, check that a user-visible offline message or fallback UI is rendered when `isConnected` is false → if missing → **warning**

### 🔗 Deep links
- Check AndroidManifest for `intent-filter` with `android.intent.action.VIEW` + custom scheme
- Check Info.plist for `CFBundleURLTypes` array
- If one platform has deep links configured and the other doesn't → warning
- Check that `android:autoVerify="true"` is set for App Links (HTTPS deep links)

### 📦 Dependencies
Read package.json and flag:
- `react-native-camera` → deprecated, migrate to `react-native-vision-camera`
- `@react-native-community/async-storage` → moved to `@react-native-async-storage/async-storage`
- `react-native-splash-screen` → deprecated, use `expo-splash-screen` or `react-native-bootsplash`
- `react-native-firebase` (non-scoped) → use `@react-native-firebase/*`
- RN version below 0.73 → warn about New Architecture and Hermes support
- Any package with `"*"` as version → error (unpinned dependency)

### 🗑️ Account deletion — Android (Play Store policy)
- Search all `.ts`, `.tsx`, `.js`, `.jsx` files for sign-up/registration signals: `signUp`, `sign_up`, `register`, `createAccount`, `create_account`, `createUser`, `SignUpScreen`, `RegisterScreen`, `onboard`
- If any sign-up signals are found:
  - Search source files for account deletion signals: `deleteAccount`, `delete_account`, `deleteUser`, `removeAccount`, `deactivateAccount`, `DeactivateScreen`, `DeleteAccountScreen`
  - If no deletion signals found → **error** (Google Play requires a deletion option for any app with account creation; violating apps are removed from the store)
  - If deletion signals found, check they are reachable from a settings or profile screen (not buried behind support-only flows) → if not reachable → **warning**
  - Check that the deletion flow also offers to delete all associated data (not just the account record) → if no data-deletion mention found alongside the delete call → **warning**
- If no sign-up signals found → **info** (no account deletion required, but confirm app truly has no registration flow)

### 🔒 Privacy Policy (COMP-003)
- Search all source files for a privacy policy link: `privacyPolicy`, `privacy-policy`, `privacy_policy`, `Privacy Policy`, `terms`, `termsOfService`
- If no privacy policy link found anywhere in source → **error** (both App Store and Play Store require a privacy policy URL to be accessible inside the app if the app collects any user data)
- If found, check it is reachable from a settings, profile, or onboarding screen (not only in the store listing) → if not reachable in-app → **warning**
- Check `app.json` or store metadata for `privacyPolicyUrl` → if missing → **warning**

### 📦 Assets & performance
- **PERF-001 — App bundle size**
  - List all files in `assets/` and `src/` larger than 1 MB (`find . -not -path "*/node_modules/*" -size +1M`)
  - If any single asset exceeds 5 MB → **warning** (large assets inflate download size and increase store review times)
  - If total assets exceed 30 MB → **warning** (flag for manual review; both stores have size soft-limits that trigger additional scrutiny)

- **PERF-002 — Unused assets**
  - List every file in `assets/` folder
  - For each file, search all source files for its filename → if not referenced anywhere → **warning** (dead assets inflate bundle size with no benefit)

### 🎨 UI safety
- **UI-002 — Error fallback UI**
  - Search source files for an `ErrorBoundary` component: `ErrorBoundary`, `componentDidCatch`, `react-error-boundary`
  - If no ErrorBoundary found wrapping the root navigator or App component → **warning** (without a boundary, any render error crashes the entire app with a blank white screen)

- **UI-003 — Loading states**
  - Search for data-fetching patterns (`useQuery`, `useSWR`, `useEffect` + `fetch`, `axios`)
  - Check that each fetching call has a corresponding loading indicator: `ActivityIndicator`, `Skeleton`, `isLoading`, `loading`, `isFetching`
  - If data fetching is found but no loading indicator detected in nearby code → **warning** (missing loading states cause perceived freezes and blank screens on slow connections)

### ⚙️ Android activity config
- `android:exported="true"` must be on MainActivity (required Android 12+)
- `android:windowSoftInputMode="adjustResize"` → warn, broken on Android 11+ edge-to-edge
- `android:allowBackup="false"` should be set
- `android:dataExtractionRules` should be set for Android 12+
- `SYSTEM_ALERT_WINDOW` permission → flag if present (requires Play Store justification)
- `WRITE_EXTERNAL_STORAGE` → flag as error (ignored on API 29+, signals old targeting)

### 🎯 Version & build numbers
- Check `versionCode` and `versionName` in `android/app/build.gradle`
- Check `CFBundleVersion` and `CFBundleShortVersionString` in Info.plist
- Check `version` in app.json / package.json
- Flag if versions are inconsistent across files
- Flag if versionCode is not an integer or if versionName doesn't follow semver

### 📱 Orientation
- Check `android:screenOrientation` on MainActivity or `orientation` in app.json
- If not set → info (defaults to both, may not be intended)
- If set, confirm it matches across app.json (Expo) and native files

### 📂 Android Target SDK & Compile SDK (Play Store policy hard-stop)
- Read `android/app/build.gradle` and extract `targetSdkVersion` and `compileSdkVersion`
- Compare `targetSdkVersion` against the current Play Store minimum:
  - For 2024/2025 releases the required minimum is **34** (Android 14); flag anything below as → **error** (Google Play blocks new uploads and updates targeting an SDK version below the current policy floor — apps that fail this check literally cannot be pushed to the store)
  - If `targetSdkVersion` is a variable reference (e.g. `targetSdkVersion rootProject.ext.targetSdkVersion`) → resolve it from `gradle.properties` or the root `build.gradle`; if it cannot be resolved → **warning** (verify the effective value is ≥ 34 before release)
- Compare `compileSdkVersion`: must be ≥ `targetSdkVersion` → if lower → **error** (build will fail)
- If either value is missing from the file → **error** (required fields for Play Store submission)

### 👵 Android back-press handling
- Search all `.ts`, `.tsx`, `.js`, `.jsx` files for `BackHandler` (from `react-native`) and `useBackHandler` (from `@react-native-community/hooks`)
- If no `BackHandler` usage is found anywhere:
  - Check if the app has a root stack navigator (`createNativeStackNavigator`, `createStackNavigator`) → if yes → **warning** (without a `BackHandler` override on the root screen, pressing the hardware Back button on the first screen exits the app immediately; users expect a "press Back again to exit" confirmation or the button to be a no-op when there is nothing to go back to)
- If `BackHandler` is found, verify:
  - A `BackHandler.addEventListener('hardwareBackPress', ...)` call exists in the root/home screen or the root navigator → if only in deep screens → **warning** (root-level handling is required to guard the exit)
  - The handler returns `true` when it consumes the event (prevents default exit) → if it returns `false` or `undefined` unconditionally → **warning**
- If no navigator found and no BackHandler found → **info** (single-screen app; back press behaviour may be intentional)

### 📝 Privacy Manifests — iOS (PrivacyInfo.xcprivacy)
Apple requires a `PrivacyInfo.xcprivacy` file (introduced in spring 2024) for any app that uses "Required Reason APIs".

- Check for the file at `ios/PrivacyInfo.xcprivacy` or inside `ios/<AppName>/PrivacyInfo.xcprivacy`
  - If the file does not exist → **error** (Apple's Transporter/Xcode upload will reject any build that calls a Required Reason API without the manifest; common APIs that trigger this include `UserDefaults`, `FileManager`, `SystemBootTime`, `diskSpace`)
- If the file exists, read it and check:
  - `NSPrivacyTracking` key: if the app uses any analytics/advertising SDK (see Step 1 scan) and this is `false` or missing → **warning** (set to `true` and declare the ATT usage category)
  - `NSPrivacyTrackingDomains` array: should list every domain used for tracking → if empty while analytics SDKs are present → **warning**
  - `NSPrivacyAccessedAPITypes` array: must list an entry for every Required Reason API used by the app or its dependencies → if array is empty → **warning** (the most common required entries are `NSPrivacyAccessedAPICategoryUserDefaults`, `NSPrivacyAccessedAPICategoryFileTimestamp`, `NSPrivacyAccessedAPICategorySystemBootTime`, `NSPrivacyAccessedAPICategoryDiskSpace`)
- Search source files and `Podfile.lock` for fingerprinting signals: `UIDevice.identifierForVendor`, `sysctlbyname`, `NSHomeDirectory`, `mach_absolute_time` → if found without a matching `NSPrivacyAccessedAPITypes` entry → **error** (Apple calls these "fingerprinting APIs" and rejects apps that call them without a declared reason)

### 🔗 ProGuard / R8 obfuscation (Android release builds)
- Read `android/app/build.gradle` and locate the `release` block inside `buildTypes`
- Check `minifyEnabled`:
  - If `minifyEnabled false` in the release block → **warning** (disabling minification/obfuscation on release builds exposes internal API logic, endpoint URLs, and token handling to anyone who decompiles the APK; it also produces a larger APK)
  - If `minifyEnabled` is missing entirely → **warning** (default is `false`; explicitly set it to `true` for release)
  - If `minifyEnabled true` → check that a `proguardFiles` entry exists pointing to a rules file (`proguard-rules.pro`) → if missing → **warning** (Hermes/RN-specific ProGuard rules are needed or the app will crash on release; the default file must be listed)
- Check `shrinkResources`:
  - If `shrinkResources false` or missing in release block → **info** (enable alongside `minifyEnabled` to strip unused resources and reduce APK size)
- If `minifyEnabled true` and `proguardFiles` exists → **passed**

### 🔐 App Store export compliance (encryption)
Apple requires every build submitted to App Store Connect to declare whether it uses non-exempt encryption. Without the declaration in `Info.plist`, reviewers must answer the compliance questions manually in App Store Connect for every single build upload.

- Read `ios/**/Info.plist` and check for the `ITSAppUsesNonExemptEncryption` key:
  - If the key is **missing** → **warning** (every build upload to App Store Connect will prompt for manual export compliance answers; add the key to automate this and eliminate the per-build friction)
  - If the key is present and set to `false` → **passed** (correct for apps that use only standard HTTPS/TLS with no custom or proprietary encryption algorithms; no further action needed)
  - If the key is present and set to `true` → **info** (app declares non-exempt encryption; confirm that an `ITSEncryptionExportComplianceCode` key is also present with the ERN number obtained from the US Bureau of Industry and Security → if missing → **warning**, the declaration is incomplete)
- Cross-check: search source files for custom encryption signals — `CryptoKit`, `CommonCrypto`, `RNCrypto`, `crypto-js`, `sjcl`, `forge`, `aes`, `rsa`, `hmac`, `pbkdf2` (excluding usage inside HTTPS/TLS libraries) → if found and `ITSAppUsesNonExemptEncryption` is `false` → **warning** (the key may be set incorrectly; custom crypto beyond standard TLS is considered non-exempt and requires an ERN)
- Note: standard HTTPS, TLS, and Apple's own frameworks (e.g. `URLSession`, `SecureTransport`) are **exempt** and justify `ITSAppUsesNonExemptEncryption = false`

### ⚡ Hermes engine verification
Hermes is the default and recommended JS engine for React Native ≥ 0.70; JSC (JavaScriptCore) should not be used in new projects.

**Android**
- Read `android/app/build.gradle`:
  - For RN < 0.70: look for `enableHermes: true` inside the `project.ext.react` block → if `false` or missing → **error**
  - For RN ≥ 0.70: Hermes is opt-out; look for an explicit `enableHermes: false` or `hermesEnabled=false` in `android/gradle.properties` → if found → **error** (Hermes has been intentionally disabled; this causes significant performance degradation on low-end Android devices)
- If Hermes is confirmed enabled (or is the default for the RN version) → **passed**

**iOS**
- Read `ios/Podfile`:
  - For RN < 0.70: look for `:hermes_enabled => true` in the `use_react_native!` block → if `false` or missing → **error**
  - For RN ≥ 0.70: look for an explicit `:hermes_enabled => false` → if found → **error**
- For Expo projects: check `app.json` / `app.config.js` for `expo-build-properties` plugin with `{ "ios": { "useHermes": false } }` → if found → **error**
- Cross-check: if Hermes is enabled on Android but disabled on iOS (or vice versa) → **warning** (inconsistent engine across platforms can cause subtle JS behaviour differences that are hard to debug)

## Step 3 — Output the report

Format the report exactly like this:

---

## RN Audit Report — [app name from package.json]

**[X errors · Y warnings · Z info · W passed]**

---

### 🔴 Errors (must fix before release)

For each error:
> **[Category]** — [specific issue with exact file, key, or value]
> 
> Fix: `exact fix with code snippet or config change`

---

### 🟡 Warnings (should fix)

For each warning:
> **[Category]** — [specific issue]
> 
> Fix: `exact fix`

---

### 🔵 Info

For each info item:
> **[Category]** — [observation]

---

### ✅ Passed

List all checks that passed cleanly as a single line each.

---

### 📋 Next steps

List the 3 most important things to fix first, in priority order, with the exact file and line to change.
