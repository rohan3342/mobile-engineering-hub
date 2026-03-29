# React Native Security: A Defense-in-Depth Architecture Guide

---

## 1. Introduction

Any React Native app that handles user accounts, sensitive data, or real-money transactions is a target. Whether you're building a banking app, a healthcare platform, an e-commerce experience, or a productivity tool with user authentication — the same vulnerabilities apply, and the consequences of a breach scale with how much your users trust you with.

### Why Mobile App Security Matters

A compromised app exposes your users and your business simultaneously. Attackers can intercept API calls, reverse-engineer business logic, bypass authentication flows, and exploit backend services at scale. In regulated industries the stakes are even higher — compliance violations, forced audits, and permanent reputational damage. But even outside regulated verticals, a single publicly disclosed breach erodes the trust that took years to build.

### What Makes React Native Apps Vulnerable

React Native apps have a few structural characteristics that attackers exploit:

- **JavaScript bundle exposure**: The JS bundle can be extracted from the APK/IPA and reverse-engineered, leaking API keys, business logic, or endpoints.
- **Native bridge attack surface**: The bridge between JS and native code can be hooked using tools like Frida or Xposed.
- **Rooted/jailbroken devices**: These bypass OS-level sandboxing, giving attackers full control over app memory, files, and network traffic.
- **No built-in backend validation**: React Native apps talk to APIs directly — nothing stops an attacker from replaying those requests outside the app.
- **Debug mode left enabled**: Apps shipped in debug mode expose internal state and disable security protections.

The diagram below maps each attack technique to the app component it targets, and the defense that mitigates it:

```mermaid
flowchart LR
   subgraph ATTACKS["Attacker Techniques"]
       direction TB
       A1["Decompile APK/IPA\nextract JS bundle"]
       A2["Script API calls\ndirectly — no app needed"]
       A3["Install custom CA\nintercept HTTPS"]
       A4["Frida / Xposed\nhook native bridge"]
       A5["Read filesystem\non rooted device"]
   end

   subgraph APP["React Native App Surface"]
       direction TB
       B1[JS Bundle]
       B2[Native Bridge]
       B3[Local Storage]
       B4[Network Layer]
       B5[Backend APIs]
   end

   subgraph DEFENSES["This Guide's Defenses"]
       direction TB
       D1[Code obfuscation\n+ env injection]
       D2[Firebase App Check]
       D3[SSL Pinning]
       D4["JailMonkey + freeRASP\nhook & integrity detection"]
       D5[Encrypted MMKV\n+ Keychain]
   end

   A1 -- leaks keys & endpoints --> B1
   A2 -- abuses directly --> B5
   A3 -- intercepts --> B4
   A4 -- hooks --> B2
   A5 -- reads --> B3

   B1 -. mitigated by .-> D1
   B5 -. mitigated by .-> D2
   B4 -. mitigated by .-> D3
   B2 -. mitigated by .-> D4
   B3 -. mitigated by .-> D5
```

### The Security Layer Model

No single tool solves mobile security. A robust solution is layered:

| Layer | Tool | Threat Addressed |
|---|---|---|
| Device Integrity | JailMonkey | Rooted/jailbroken devices, emulators, debug mode |
| Runtime Application Self-Protection | freeRASP *(free tier: 100k download cap — see Section 3)* | App tampering detection, hook detection, unofficial stores, screen capture blocking, obfuscation bypass, VPN / time / location spoofing |
| Backend Abuse Protection | Firebase App Check | Unauthorized API/database access, bot traffic |
| Transaction Fraud Prevention | Sardine SDK *(fintech)* | Fraud, account takeover, risky onboarding |
| Data at Rest & Authentication | MMKV + Keychain + Biometrics | Credential theft, storage extraction, unattended access |
| Network Transport | SSL Pinning + Payload Encryption | MITM attacks, TLS interception, in-transit data exposure |

Each layer addresses a different vector. Together, they form a defense-in-depth strategy that significantly raises the cost and complexity of attacks against your app.

### OWASP Mobile Top 10 Coverage

This guide's tooling directly addresses six of the [OWASP Mobile Top 10 (2024)](https://owasp.org/www-project-mobile-top-10/) risk categories. Understanding the mapping clarifies *why* each tool exists, not just *what* it does:

| OWASP Category | Risk | This Guide's Defense |
|---|---|---|
| M1 — Improper Credential Usage | Hardcoded secrets, insecure credential storage in plaintext | Keychain + Biometrics (Section 6); env-var injection at build time (Section 8) |
| M2 — Inadequate Supply Chain Security | Tampered dependencies, repackaged or resigned binaries | freeRASP `appIntegrity` callback (Section 3) |
| M3 — Insecure Authentication / Authorization | Weak or bypassable authentication flows | Biometrics + Keychain `BIOMETRY_ANY` access control (Section 6) |
| M5 — Insecure Communication | MITM attacks, plaintext traffic, TLS misconfiguration | SSL Pinning + Payload Encryption (Section 7) |
| M8 — Security Misconfiguration | Debug mode in production builds, exposed API keys in JS bundle | JailMonkey `isDebuggedMode`; ProGuard/Hermes obfuscation; `__DEV__` guards |
| M9 — Insecure Data Storage | Unencrypted tokens, plaintext credentials in AsyncStorage | Encrypted MMKV + Keychain (Section 6) |

M4 (Insufficient Input/Output Validation), M6 (Inadequate Privacy Controls), M7 (Insufficient Binary Protections), and M10 (Insufficient Cryptography) fall outside this guide's scope but remain equally important for a complete production security posture. M7 is partially mitigated by freeRASP's obfuscation detection callback, which flags when your Android release build was shipped without ProGuard.

---

## 2. JailMonkey — Device Integrity Checks

>  JailMonkey runs native checks at app startup to detect rooted/jailbroken devices, emulators, and runtime hooks. The result is a single boolean you gate your app on. It is a **client-side risk signal, not a guarantee** — a sophisticated attacker can bypass it, which is why you send the signal to your backend and combine it with server-side enforcement.

### What JailMonkey Does

[JailMonkey](https://github.com/GantMan/jail-monkey) is a React Native library that performs runtime device integrity checks. It detects whether a device is **rooted (Android)**, **jailbroken (iOS)**, running on an **emulator**, in **debug mode**, has **ADB enabled**, uses **mock location**, or shows signs of **runtime hooking or code injection** (e.g., Frida, Xposed).

All checks are performed **client-side** using native code bridges. There is no network activity, no data collection, and no sensitive permissions required — the source code is fully auditable on GitHub.

### How It Works Internally

**On Android**, JailMonkey integrates [RootBeer](https://github.com/scottyab/rootbeer), which performs multiple low-level checks:

- Presence of root management apps (SuperSU, Magisk)
- Existence of the `su` binary
- Dangerous system properties and writable system paths
- Build tag test-keys (common in custom ROMs)
- Magisk binary detection
- ADB enabled status
- App installed on external storage
- Mock location enabled
- Emulator signatures (device IDs, files, properties)
- Runtime hooking indicators

**On iOS**, JailMonkey checks:

- Existence of jailbreak artifacts (`/Applications/Cydia.app`, `apt`, etc.)
- Ability to write outside the app sandbox
- Presence of jailbreak tools loaded at runtime
- Simulator detection
- Debug mode and hooking/injection frameworks

### Why Device Integrity Checks Matter

On a rooted or jailbroken device, the OS security model breaks down entirely. An attacker can:

- Read your app's private keychain/keystore data
- Intercept and modify network traffic even over HTTPS (SSL pinning bypass)
- Hook your app's functions at runtime using tools like Frida
- Dump memory to extract tokens or secrets
- Bypass biometric and PIN checks

JailMonkey lets you detect these conditions before your app does anything sensitive.

```mermaid
flowchart TD
   A([App Startup]) --> B{__DEV__ mode?}
   B -- Yes --> C([Skip checks\nDev environment])
   B -- No --> D[Run integrity checks]
   D --> E{isJailBroken or\ntrustFall?}
   E -- Yes --> FAIL
   E -- No --> F{hookDetected?}
   F -- Yes --> FAIL
   F -- No --> G{ADB enabled?}
   G -- Yes --> FAIL
   G -- No --> H{isDebuggedMode?}
   H -- Yes --> FAIL
   H -- No --> I{RootBeer flags\nany truthy?}
   I -- Yes --> FAIL
   I -- No --> PASS
   PASS([isDeviceSecure = true\nProceed normally])
   FAIL([isDeviceSecure = false\nBlock or restrict features])
```

### Installation

```bash
npm install jail-monkey
# or
yarn add jail-monkey
```

For iOS, run pod install:

```bash
cd ios && pod install
```

### Code Example

A clean pattern is to encapsulate all checks in a single `useDeviceSecurity` hook that returns a boolean. Everything in the app keys off that one value.

```ts
import { useEffect, useState } from "react";
import JailMonkey from "jail-monkey";

const useDeviceSecurity = (): boolean => {
 const [isDeviceSecure, setIsDeviceSecure] = useState(true);

 useEffect(() => {
   if (__DEV__) return; // skip checks in development

   const checkSecurity = async () => {
     const rootedDetection = JailMonkey.androidRootedDetectionMethods;
     const isRootedByRootBeer = rootedDetection?.rootBeer
       ? Object.values(rootedDetection.rootBeer).some(Boolean)
       : false;

     const isCompromised =
       JailMonkey.isJailBroken()      ||
       JailMonkey.trustFall()         ||
       JailMonkey.hookDetected()      ||
       JailMonkey.AdbEnabled?.()      ||
       (await JailMonkey.isDebuggedMode?.()) ||
       isRootedByRootBeer             ||
       rootedDetection?.jailMonkey;

     setIsDeviceSecure(!isCompromised);
   };

   checkSecurity();
 }, []);

 return isDeviceSecure;
};
```

A few points worth noting:

- **`__DEV__` bypass**: Checks are skipped in development so you're never blocked on a simulator.
- **`trustFall()`**: A convenience aggregator covering several checks in one call.
- **`androidRootedDetectionMethods`**: Gives access to both RootBeer's granular flags and JailMonkey's own detector — checking both improves Android coverage.
- **Optional chaining (`?.`)**: Some APIs are Android-only; the optional chaining prevents crashes on iOS.

At startup, consume the hook and route accordingly:

```tsx
const isDeviceSecure = useDeviceSecurity();

// In your startup navigation logic:
if (!isDeviceSecure && !__DEV__) navigate('ApplicationUnavailable');
```

### Important Caveat: Client-Side Limitations

JailMonkey's checks run entirely on the device. A sufficiently advanced attacker can patch or bypass them. This is a **first line of defense**, not a complete solution. Its real value is:

- Blocking unsophisticated attacks and casual jailbreakers
- Sending device integrity signals to your backend for risk scoring
- Satisfying compliance requirements around device posture

Treat the results as **risk signals**, not absolute truth. Combine them with server-side enforcement (which is where Firebase App Check comes in).

The diagram below shows how JailMonkey fits into a layered trust decision — the client-side block stops casual attackers, but backend enforcement is the authoritative layer:

```mermaid
flowchart TD
   STARTUP([App Startup]) --> JM{JailMonkey\nchecks pass?}

   JM -- No --> RESTRICT["Block or restrict\napp features client-side"]
   JM -- Yes --> ALLOW[Allow normal app flow]
   JM -- Either result --> SIGNAL["Send device_integrity signal\nto backend on every auth request"]

   ALLOW --> APPCHECK[Firebase App Check\ntoken on every request]
   APPCHECK --> BACKEND

   SIGNAL --> BACKEND[("Your Backend\nRisk Engine")]

   BACKEND --> DECISION{Risk\ndecision}
   DECISION -- Low risk --> APPROVE([Approve request])
   DECISION -- High risk --> DENY([Deny or step-up challenge])
```

> **Key principle**: The client block (left path) stops unsophisticated attackers immediately. But your backend receiving `device_integrity=false` as a signal is what gives you audit trails, rate limits, and risk-based authentication — regardless of what the client claims.

### Practical Scenarios

| Scenario | JailMonkey Response | Backend Signal |
|---|---|---|
| Banking app on rooted device | Block account access UI | `device_integrity=false` sent for audit log |
| Crypto wallet with Frida hook detected | Refuse to show seed phrase | `hook_detected=true` flags session as high-risk |
| KYC identity verification | Warn user before initiating | Integrity status forwarded to fraud backend |
| Emulator detected in production | Block registration flow | Emulator flag triggers manual review queue |

### Structuring Device Signals for Your Backend

When JailMonkey detects a compromised state, don't just block the user client-side — propagate structured signals to your backend so your risk engine can make authoritative decisions. This is what transforms a client-side boolean into an auditable, enforceable record.

A well-structured payload captures all relevant flags atomically with a timestamp:

```ts
import { Platform } from 'react-native';
import JailMonkey from 'jail-monkey';

interface DeviceIntegrityPayload {
 timestamp: string;          // ISO 8601
 platform: 'ios' | 'android';
 signals: {
   isJailBroken: boolean;
   trustFall: boolean;
   hookDetected: boolean;
   adbEnabled: boolean | null;           // Android only
   isDebuggedMode: boolean;
   isOnExternalStorage: boolean | null;  // Android only
   mockLocation: boolean | null;         // Android only
   rootBeerFlags?: Record<string, boolean>;
 };
 isCompromised: boolean; // aggregate: any signal truthy
}

const buildDeviceIntegrityPayload = async (): Promise<DeviceIntegrityPayload> => {
 const rootedDetection = JailMonkey.androidRootedDetectionMethods;

 const signals = {
   isJailBroken:         JailMonkey.isJailBroken(),
   trustFall:            JailMonkey.trustFall(),
   hookDetected:         JailMonkey.hookDetected(),
   adbEnabled:           JailMonkey.AdbEnabled?.() ?? null,
   isDebuggedMode:       (await JailMonkey.isDebuggedMode?.()) ?? false,
   isOnExternalStorage:  JailMonkey.isOnExternalStorage?.() ?? null,
   mockLocation:         JailMonkey.isMockingLocation?.() ?? null,
   rootBeerFlags:        rootedDetection?.rootBeer ?? undefined,
 };

 return {
   timestamp:    new Date().toISOString(),
   platform:     Platform.OS as 'ios' | 'android',
   signals,
   isCompromised: Object.values(signals).some(v => v === true),
 };
};
```

Send this payload on every authenticated request — not just at startup. A device can be rooted while the app is running, or root-hiding tools can fail mid-session. Attaching the integrity payload as a signed header claim gives your backend continuous visibility and an audit trail per request.

```mermaid
sequenceDiagram
   participant App as React Native App
   participant JM as JailMonkey
   participant API as Your Backend Risk Engine

   App->>JM: Run all checks (async)
   JM-->>App: Aggregate signals

   App->>API: Authenticated request\n+ X-Device-Integrity: {base64 JSON payload}
   Note over API: Decode and validate payload
   alt isCompromised === false
       API-->>App: Normal response
   else isCompromised === true
       API->>API: Log event · flag session · apply step-up
       API-->>App: 403 or step-up challenge
   end
```

---

## 3. freeRASP — Full Runtime Application Self-Protection

> freeRASP is a **Runtime Application Self-Protection (RASP)** library by [Talsec](https://www.talsec.app) that significantly extends beyond JailMonkey's device integrity checks. It detects rooted/jailbroken devices, runtime hooks (Frida/Xposed), debugger attachment, emulators, unofficial app stores, **app integrity tampering** (repackaged or modified builds), obfuscation bypass, screen capture, VPN usage, time/location spoofing, and more — all through a single React Native hook with a unified callback API and a kill-on-bypass enforcement mode.

> ⚠️ **Pricing caveat — read before integrating**: freeRASP is **freemium** software governed by a [Fair Usage Policy (FUP)](https://docs.talsec.app/freerasp/fair-usage-policy-fup). The free tier is **capped at 100,000 app downloads**. Beyond that threshold, you must upgrade to the paid **RASP+** plan. RASP+ also removes Talsec data collection, replaces the universal free binary with an **app-specific hardened binary** (significantly harder to bypass), and adds the **AppiCrypt®** API integrity cryptogram. For a banking or fintech app with a real user base, treat RASP+ as the realistic production target and budget for it accordingly.

### freeRASP vs JailMonkey

freeRASP is a strict superset of what JailMonkey provides. Both detect rooting, jailbreaking, emulators, and hook frameworks. freeRASP also adds:

| Capability | JailMonkey | freeRASP |
|---|---|---|
| Root / Jailbreak detection | ✅ | ✅ |
| Emulator detection | ✅ | ✅ |
| Hook framework detection (Frida, Xposed) | ✅ | ✅ |
| Debug mode detection | ✅ | ✅ |
| ADB enabled detection | ✅ | ✅ |
| **App integrity verification** (tamper detection) | ❌ | ✅ |
| **Unofficial store detection** | ❌ | ✅ |
| **Obfuscation issue detection** | ❌ | ✅ |
| **Screen capture / screen recording detection + blocking** | ❌ | ✅ |
| **System VPN detection** | ❌ | ✅ |
| **Time spoofing detection** | ❌ | ✅ |
| **Location spoofing detection** (Android) | ❌ | ✅ |
| Unsecure Wi-Fi detection (Android) | ❌ | ✅ |
| Automation framework detection (Android) | ❌ | ✅ |
| Multi-instance detection (Android) | ❌ | ✅ |
| **Kill-on-bypass enforcement** | ❌ | ✅ |
| Malware detection (add-on module) | ❌ | ✅ |
| Security dashboard portal (Talsec Portal) | ❌ | ✅ |

The two critical additions over JailMonkey are **app integrity verification** — detecting whether your APK/IPA has been repackaged, resigned, or tampered with before reaching the user — and **kill-on-bypass**, which terminates the app process automatically if it detects an attacker manipulating or hooking the RASP threat callback mechanism itself.

If your app already uses JailMonkey, freeRASP can replace it entirely — the root/jailbreak coverage is equivalent, and you gain the broader threat surface at the cost of the download cap.

### Installation

```bash
npm install freerasp-react-native
# or
yarn add freerasp-react-native
```

For iOS, run pod install:

```bash
cd ios && pod install
```

**Android prerequisites** — freeRASP requires `minSdkVersion` ≥ 23 (Android 6.0) and Kotlin ≥ 2.0.0. In `android/build.gradle`:

```groovy
buildscript {
   ext {
       minSdkVersion = 23        // required minimum
       kotlinVersion = '2.0.0'   // required since freeRASP 4.0.0
   }
   dependencies {
       classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:2.0.0")
   }
}
```

**Android — screen capture detection** (optional, requires Android 14+ / 15+):

Add to `AndroidManifest.xml` inside the `<manifest>` root tag:

```xml
<uses-permission android:name="android.permission.DETECT_SCREEN_CAPTURE" />
<uses-permission android:name="android.permission.DETECT_SCREEN_RECORDING" />
```

### Configuration

Initialize freeRASP at your app entry point using the `useFreeRasp` hook. Get your Android signing certificate hash by following Talsec's [signing certificate guide](https://docs.talsec.app/freerasp/wiki/getting-signing-certificate-hash).

```ts
import { useFreeRasp } from 'freerasp-react-native';

const config = {
 androidConfig: {
   packageName: 'com.yourapp',
   certificateHashes: ['YOUR_BASE64_SIGNING_CERT_HASH='], // release signing cert hash(es)
   supportedAlternativeStores: ['com.sec.android.app.samsungapps'],
 },
 iosConfig: {
   appBundleId: 'com.yourapp',
   appTeamId: 'YOUR_APPLE_TEAM_ID',
 },
 watcherMail: 'security@yourcompany.com', // receives Talsec Portal reports and SDK updates
 isProd: !__DEV__,    // false in development — skips production checks on simulator
 killOnBypass: true,  // terminate the app if RASP callbacks are hooked or tampered with
};
```

Key configuration notes:

- **`isProd`**: Set to `false` in development so you are not blocked on a simulator. In production, all checks run. See [Talsec's `isProd` documentation](https://docs.talsec.app/freerasp/wiki/isprod-flag).
- **`killOnBypass: true`**: The SDK will terminate the process if it detects an attacker manipulating the threat callback mechanism. This is the anti-tamper enforcement layer on top of detection.
- **`watcherMail`**: Required. Used for Talsec Portal access and security report delivery.

### Threat Detection and Reactions

Define a callback object mapping each detected threat to a response. At minimum, log the event and send a risk signal to your backend. For the highest-severity threats in a financial app, block or terminate:

```ts
const actions = {
 // ── High severity: block app access and signal backend ─────────────────────
 privilegedAccess: () => {
   // Rooted (Android) or jailbroken (iOS)
   sendRiskSignalToBackend('privileged_access');
   setIsDeviceSecure(false);
 },
 hooks: () => {
   // Frida, Xposed, or other hook framework detected
   sendRiskSignalToBackend('hook_detected');
   setIsDeviceSecure(false);
 },
 appIntegrity: () => {
   // APK/IPA has been tampered with or repackaged
   sendRiskSignalToBackend('app_integrity_fail');
   setIsDeviceSecure(false);
 },
 debug: () => {
   // Debugger attached or debug mode active
   sendRiskSignalToBackend('debug_mode');
   setIsDeviceSecure(false);
 },

 // ── Medium severity: flag session, send backend signal ──────────────────────
 simulator: () => {
   sendRiskSignalToBackend('simulator');
   if (!__DEV__) setIsDeviceSecure(false);
 },
 unofficialStore: () => {
   // App was not installed from an official store — possible repackaging
   sendRiskSignalToBackend('unofficial_store');
 },
 obfuscationIssues: () => {
   // Android only — app was shipped without obfuscation, leaking business logic
   sendRiskSignalToBackend('obfuscation_issues');
 },

 // ── Runtime environment signals ─────────────────────────────────────────────
 systemVPN: () => sendRiskSignalToBackend('vpn_active'),
 devMode: () => sendRiskSignalToBackend('dev_mode'),         // Android
 adbEnabled: () => sendRiskSignalToBackend('adb_enabled'),   // Android
 timeSpoofing: () => sendRiskSignalToBackend('time_spoofing'),
 locationSpoofing: () => sendRiskSignalToBackend('location_spoofing'), // Android

 // ── Screen capture — block for sensitive screens (account details, KYC) ─────
 screenshot: () => console.warn('Screenshot taken on sensitive screen'),
 screenRecording: () => console.warn('Screen recording active on sensitive screen'),

 // ── Device state signals ────────────────────────────────────────────────────
 passcode: () => sendRiskSignalToBackend('no_passcode_set'),
 secureHardwareNotAvailable: () => sendRiskSignalToBackend('no_secure_hardware'),
 deviceBinding: () => sendRiskSignalToBackend('device_binding_fail'),
 deviceID: () => sendRiskSignalToBackend('device_id_fail'), // iOS only

 // ── Android-only signals ────────────────────────────────────────────────────
 unsecureWifi: () => sendRiskSignalToBackend('unsecure_wifi'),
 automation: () => sendRiskSignalToBackend('automation_detected'),
 multiInstance: () => sendRiskSignalToBackend('multi_instance'),
};

// Optional: callback when all initial startup checks are complete
const raspExecutionStateActions = {
 allChecksFinished: () => {
   console.log('freeRASP initial checks complete');
 },
};

// Start freeRASP — call outside useEffect, at component top level
// freeRASP runs continuous periodic checks throughout the app lifecycle
useFreeRasp(config, actions, raspExecutionStateActions);
```

> **Important**: `useFreeRasp` must be called at the component top level, **not** inside `useEffect`. freeRASP performs continuous checks throughout the session — not just at startup.

### Proactive Screen Capture Blocking

Beyond detecting screenshots, freeRASP can **actively prevent** screen capture on sensitive screens such as account numbers, card details, KYC documents, and transaction confirmations:

```ts
import { blockScreenCapture, isScreenCaptureBlocked } from 'freerasp-react-native';

// Block screen capture when a sensitive screen mounts, restore on unmount
useEffect(() => {
 blockScreenCapture(true);
 return () => blockScreenCapture(false);
}, []);
```

### Detection Flow

```mermaid
flowchart TD
   A([App Startup]) --> B{isProd?}
   B -- No / dev --> C([Checks skipped\nDev environment])
   B -- Yes --> D[freeRASP initializes\ncontinuous periodic checks]
   D --> E{privilegedAccess\nhooks or appIntegrity?}
   E -- Yes --> BLOCK[Block app\nsend risk signal to backend]
   E -- No --> F{simulator or\nunofficialStore?}
   F -- Yes in prod --> BLOCK
   F -- No --> G{debug or\nobfuscationIssues?}
   G -- Yes --> RISK[Flag session\nsend backend signal]
   G -- No --> I([App proceeds normally\nfreeRASP monitors continuously])
   BLOCK --> J{killOnBypass\nenabled?}
   J -- Yes --> KILL([Process terminated])
   J -- No --> RESTRICT([Restrict features])
```

### Pricing and Free Tier Limitations

| | freeRASP (Free) | RASP+ (Paid) |
|---|---|---|
| Download limit | **100,000** | Unlimited |
| Data collection | Sent to Talsec database | Your infrastructure (or fully disabled) |
| SDK binary | Universal — bypass scripts are more reusable | App-specific hardened binary |
| AppiCrypt® API protection | ❌ | ✅ |
| Enterprise SLA / support | Community | Enterprise SLA |
| Fintech compliance features | Limited | Full |

**Recommendation for production fintech apps**:

- **Development and early testing**: the free tier is fine.
- **Production at scale (> 100k downloads)**: budget for **RASP+** upfront. The free binary is also a universal binary — the same build is shared across all free-tier integrators, making community-developed bypass scripts more broadly applicable. RASP+ generates app-specific hardened binaries, raising the effort required to reverse-engineer or bypass significantly.
- **If RASP+ cost is not yet justified**: JailMonkey has no download cap, no data collection, and covers the core device integrity signals. Keep JailMonkey as a baseline until you are ready to commit to freeRASP RASP+.

### Migrating from JailMonkey to freeRASP

If your app already uses JailMonkey, the migration is straightforward — freeRASP is a functional superset and every JailMonkey check maps directly to a freeRASP callback.

| JailMonkey check | freeRASP callback | Notes |
|---|---|---|
| `isJailBroken()` / `trustFall()` | `privilegedAccess` | Equivalent coverage |
| `hookDetected()` | `hooks` | Equivalent coverage |
| `isDebuggedMode()` | `debug` | Equivalent coverage |
| `AdbEnabled()` | `adbEnabled` | Android only |
| `isMockingLocation()` | `locationSpoofing` | Android only |
| *(no equivalent)* | `appIntegrity` | **New** — detects tampered or repackaged builds |
| *(no equivalent)* | `unofficialStore` | **New** — detects sideloaded installs |
| *(no equivalent)* | `killOnBypass` | **New** — process termination if RASP is bypassed |

**Migration steps**:

1. Remove `jail-monkey` from your dependencies.
2. Install `freerasp-react-native` and run `pod install`.
3. Replace the `useDeviceSecurity` hook body with `useFreeRasp` initialization.
4. Map each previous `JailMonkey` check to the corresponding `actions` callback (table above).
5. Add `appIntegrity` and `unofficialStore` callbacks with appropriate backend signals.
6. Confirm `isProd: !__DEV__` and `killOnBypass: true` for production builds.
7. Submit your signing certificate hash and bundle details per Talsec's [setup guide](https://docs.talsec.app/freerasp/wiki/getting-signing-certificate-hash).

> Your `sendRiskSignalToBackend` utility function remains unchanged — it is called identically from both JailMonkey and freeRASP callbacks.

---

## 4. Firebase App Check — Backend Abuse Protection

>  App Check issues cryptographically signed **attestation tokens** proving a request came from your real, unmodified app on a real device. Your backend middleware rejects everything without a valid token — bots and scripts cannot obtain one from outside the app, so they never reach your business logic.

### The Problem It Solves

Even if your app is secure, your **backend APIs are exposed to the internet**. Nothing stops an attacker from extracting your API endpoints from the JS bundle and calling them directly — bypassing the app entirely. This enables:

- Credential stuffing attacks against your auth endpoints
- Scraping your database through your own API
- Abusing Cloud Functions for spam or DDoS
- Creating fake accounts at scale via automation

Firebase App Check solves this by ensuring **only your legitimate app** can call your backend resources.

### How App Check Works

App Check issues a short-lived **attestation token** that your app must send with every request. Your backend verifies this token with Firebase before processing the request. Tokens are generated using platform-specific attestation providers:

| Platform | Provider | What It Verifies |
|---|---|---|
| iOS | App Attest (DeviceCheck fallback) | Cryptographic proof from Apple that request comes from a genuine, unmodified app |
| Android | Play Integrity (SafetyNet fallback) | Google's verdict on app integrity and device safety |
| Web | reCAPTCHA Enterprise / v3 | Bot detection |
| Debug | Debug provider | Local development only — never ship to production |

The attestation token is opaque to your app — Firebase handles the validation. Your backend simply enforces that valid tokens must be present.

```mermaid
sequenceDiagram
   participant App as React Native App
   participant SDK as App Check SDK
   participant Attest as Play Integrity / App Attest
   participant FB as Firebase Servers
   participant API as Your Backend API

   App->>SDK: initializeAppCheck(provider)
   SDK->>Attest: Request signed attestation
   Attest-->>SDK: Signed attestation blob
   SDK->>FB: Exchange attestation for token
   FB-->>SDK: Short-lived App Check token
   Note over SDK,App: Token cached & auto-refreshed

   App->>API: Request + X-Firebase-AppCheck header
   API->>FB: Admin SDK verifyToken(token)
   FB-->>API: Valid / Invalid
   alt Token valid
       API-->>App: 200 OK
   else Token missing or invalid
       API-->>App: 401 Unauthorized
   end
```

### What It Protects

- **Firebase Realtime Database** and **Firestore** — enforce App Check in security rules
- **Cloud Functions** — check tokens in function middleware
- **Cloud Storage** — restrict read/write to attested apps
- **Custom backends** — verify tokens manually using the Firebase Admin SDK
