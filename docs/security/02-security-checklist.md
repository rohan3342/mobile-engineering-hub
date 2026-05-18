# React Native Security — Pre-Launch Checklist

Use this as a gate before shipping to production. Every item maps back to the [Defense-in-Depth Implementation Guide](./01-rn-security-defense-in-depth.md) — follow the link if you need context on why an item exists.

---

## Device Integrity

- [ ] JailMonkey or freeRASP initialized at app startup — not inside `useEffect`
- [ ] `__DEV__` / `isProd: !__DEV__` guards prevent blocking in development and CI
- [ ] Structured device integrity signals sent to backend on every authenticated request
- [ ] Compromised device state blocks or restricts sensitive app features client-side

## Runtime Protection (freeRASP)

- [ ] `appIntegrity` callback configured with your release signing certificate hashes
- [ ] `killOnBypass: true` set in production configuration
- [ ] `watcherMail` configured to receive Talsec Portal security reports
- [ ] App download count tracked against the 100,000 FUP cap; RASP+ upgrade budgeted if needed

## Firebase App Check

- [ ] Production providers configured: Play Integrity (Android), App Attest (iOS)
- [ ] Debug tokens stored in `.env` (gitignored) — not committed to source control
- [ ] Backend middleware enforces App Check on all sensitive routes before any handler runs
- [ ] Play Integrity token retrieval includes retry logic for cold-start transient failures
- [ ] `isTokenAutoRefreshEnabled: true` in `initializeAppCheck`

## SSL Pinning

- [ ] Both active certificate pin AND backup pin embedded
- [ ] Pinning disabled in `__DEV__` to allow proxy tooling during development
- [ ] Certificate rotation runbook documented and tested with the backup pin
- [ ] SPKI (public key) hash used in preference to full leaf certificate hash
- [ ] Emergency kill-switch (remote config, App Check protected) in place for forced rotation

## Data at Rest

- [ ] `AsyncStorage` not used for any security-relevant data
- [ ] Encrypted MMKV (`new MMKV({ encryptionKey })`) used for session metadata and device identifiers
- [ ] MMKV encryption key generated on first launch, stored in Keychain — never hardcoded in the bundle
- [ ] Credentials stored in Keychain with `BIOMETRY_ANY` + `WHEN_UNLOCKED` access control
- [ ] `resetGenericPassword` called on logout or session expiry
- [ ] AsyncStorage → MMKV migration completed (if applicable)

## Payload Encryption *(if applicable)*

- [ ] Server public key fetched from an App Check + SSL-pinned endpoint at startup
- [ ] Public key cached in encrypted MMKV — never hardcoded in the JS bundle
- [ ] AES-256-GCM used with a unique per-request IV and GCM auth tag
- [ ] Applied selectively to highest-sensitivity endpoints, not globally

## Sardine SDK *(fintech apps only)*

- [ ] SDK initialized with a feature-flag kill-switch for instant remote rollback
- [ ] UUID session key injected as `X-Sardine-Session-Key` on all requests
- [ ] `trackPage`, `trackTextChange`, `trackFocusChange` instrumented on all key flows
- [ ] `submitData()` called before backend decision requests (payment/onboarding)
- [ ] Backend maps all four risk levels (`low`, `medium`, `high`, `very_high`) to distinct actions

## Testing & Validation

### Device Integrity & Runtime Protection
- [ ] JailMonkey detection verified on a physical rooted (Android) and jailbroken (iOS) device
- [ ] freeRASP `privilegedAccess` callback confirmed on a rooted test device
- [ ] freeRASP `hooks` callback confirmed using Frida (`frida -U -l test-hook.js -f com.yourapp`)
- [ ] freeRASP `appIntegrity` callback confirmed by sideloading a repackaged + re-signed APK
- [ ] Backend receives and logs `device_integrity=false` signal on compromised device

### Firebase App Check
- [ ] Requests without `X-Firebase-AppCheck` header return 401
- [ ] Requests with malformed or expired token return 401
- [ ] Play Integrity cold-start retry logic verified on fresh Android install

### SSL Pinning
- [ ] MITM proxy test passed: request fails with `SSLPinningError` when proxy CA is installed
- [ ] Backup pin rotation test passed: API calls succeed with only the backup pin active

### Sardine *(fintech apps only)*
- [ ] Behavioral signals (`trackPage`, `trackTextChange`, `trackFocusChange`) visible in Sardine sandbox API response
- [ ] All four risk levels (`low`, `medium`, `high`, `very_high`) trigger correct backend actions in staging

### CI / Automated Regression
- [ ] Unit tests mock JailMonkey and assert `isDeviceSecure` hook behaves correctly
- [ ] ESLint `no-restricted-imports` rule bans `@react-native-async-storage/async-storage`
- [ ] Release build pipeline confirms ProGuard and Hermes are enabled before signing

## Build Configuration

- [ ] No API keys, secrets, or internal URLs hardcoded in the JS bundle
- [ ] `.env` files are gitignored; a `.env.example` with placeholder values is committed
- [ ] All CI/CD secrets stored in GitHub Secrets (or equivalent) — never in source code
- [ ] GitHub Environments used to scope staging vs. production secrets separately
- [ ] Signing keystore passwords, App Store Connect keys, and match passwords are GitHub Secrets
- [ ] Stripe / payment secret keys are server-side only — never injected into the bundle
- [ ] The "decisive test" applied to every injected value: would extraction cause harm? If yes → BFF
- [ ] ProGuard / R8 obfuscation enabled on Android release builds
- [ ] Hermes engine enabled (bytecode compilation adds an obfuscation layer)
- [ ] Debug mode confirmed disabled in production builds (`__DEV__ === false`)
- [ ] CI/CD pipeline enforces production-only debug tokens never reach app store builds