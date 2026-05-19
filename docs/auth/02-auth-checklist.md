# React Native Auth — Pre-Launch Checklist

Use this as a gate before shipping to production. Every item maps back to the [Authentication Architecture Guide](./01-rn-auth-architecture.md) — follow the link if you need context on why an item exists.

---

## Auth0 Configuration

- [ ] Application type set to **Native** (not SPA) in Auth0 dashboard
- [ ] Refresh Token Rotation enabled
- [ ] Refresh Token absolute expiration configured (7–30 days)
- [ ] Token Endpoint Authentication Method set to **None** (PKCE — no client secret)
- [ ] Allowed Callback and Logout URLs configured for your app scheme (`com.yourapp://`)
- [ ] Custom claims Action deployed — roles, KYC status, account tier injected via namespace
- [ ] Auth0 Audience matches the identifier configured on your backend resource server
- [ ] `com.yourapp://auth0/callback` URL scheme registered in `AndroidManifest.xml` intent filter
- [ ] `com.yourapp://auth0/callback` URL scheme registered in `Info.plist` CFBundleURLSchemes
- [ ] `openURL` handler added to `AppDelegate.mm`
- [ ] Callback URL wiring verified: `adb shell am start` / `xcrun simctl openurl` test passes
- [ ] Android back button during login handled — app returns to login screen without hanging

## Environment Configuration

- [ ] Separate Auth0 tenants for dev, staging, and production — not one tenant with different env vars
- [ ] `.env.development`, `.env.staging`, `.env.production` created and gitignored
- [ ] `.env.example` committed with placeholder values for all required variables
- [ ] GitHub Environments used to scope secrets — staging jobs never see production credentials
- [ ] Each environment has its own Google/Apple social connection credentials
- [ ] `inspectCurrentToken()` utility present for dev debugging — confirmed stripped in prod (`__DEV__` guard)

## MFA

- [ ] MFA enabled in Auth0 dashboard (Security → Multi-factor Auth)
- [ ] MFA policy set to "Always" for banking & payments apps — "Allow Remember Browser" disabled
- [ ] Auth0 Action deployed to enforce MFA for banking & payments users or high-risk logins
- [ ] `mfa_enrollment_required` error redirects to enrollment flow — does not log user out
- [ ] PSD2 / SCA compliance verified if app processes payments (MFA on payment initiation)

## Auth Error Handling

- [ ] `AuthError` type with all error codes defined — centralised, not scattered inline
- [ ] `handleAuthError` checks network state before classifying errors
- [ ] `service_unavailable` (503) never triggers logout — shows degraded state message
- [ ] `invalid_grant` triggers full logout with "session ended on another device" message
- [ ] `user_cancelled` and `auth_timeout` are silent — app stays on login screen
- [ ] `biometric_not_enrolled` redirects to device Settings — does not lock user out
- [ ] `biometric_lockout` shows PIN fallback message — does not terminate session
- [ ] Unknown errors logged to error tracker (Sentry / Datadog) before showing generic message

## Token Strategy — Bearer Tokens (General Apps)

- [ ] Access token lifetime set to 15 minutes (or 5–10 min for banking & payments apps)
- [ ] Tokens stored in Keychain with `WHEN_UNLOCKED_THIS_DEVICE_ONLY` — never in AsyncStorage
- [ ] MMKV encryption key sourced from Keychain — not hardcoded
- [ ] Proactive refresh fires 60s before expiry
- [ ] Reactive refresh (401 interceptor) uses queue pattern — concurrent requests handled without double-refresh
- [ ] Refresh token revoked at Auth0 on logout
- [ ] Token family invalidation (reuse detection) verified in staging

## Token Strategy — HttpOnly Cookie Session (Banking & Payments Apps)

- [ ] Login response sets `HttpOnly`, `Secure`, `SameSite=Strict` cookie
- [ ] `rolling: true` configured on session middleware — every request extends expiry
- [ ] Session TTL set to 15 minutes of inactivity
- [ ] `/auth/session-refresh` requires biometric proof before issuing new cookie
- [ ] Old session invalidated (regenerated) on every `/session-refresh` call
- [ ] `@react-native-cookies/cookies` configured — `withCredentials: true` on API client
- [ ] `Access-Control-Allow-Credentials: true` and explicit `Access-Control-Allow-Origin` set on backend
- [ ] Session store is Redis (not in-memory) — survives server restarts

## Social Login

- [ ] Google Sign-In uses Authorization Code + PKCE — implicit flow not used anywhere
- [ ] Apple Sign In implemented on iOS — mandatory when any social login is offered
- [ ] Social provider tokens exchanged with Auth0 before reaching your resource server — social tokens never sent to your backend directly
- [ ] Apple `fullName` persisted to backend on first login (Apple only provides it once)
- [ ] Social login tested on physical devices — not just simulators

## OTP / Passwordless

- [ ] Auth0 Passwordless connection configured (SMS via Twilio or Email magic link)
- [ ] OTP rate limiting: max 5 requests per 15 min per user (not just per IP)
- [ ] OTP resend rate limiting: max 1 resend per minute
- [ ] Step-up challenge tokens are short-lived (5 min max) and single-use
- [ ] Step-up required for: transfers, viewing account numbers, changing security settings (banking & payments)
- [ ] OTP brute-force lockout: account locked after 3 failed attempts with cooldown
- [ ] SMS auto-read (`react-native-otp-verify`) tested on Android physical device

## Biometric Gate

- [ ] Biometrics implemented as a **presence gate** — backed by Keychain credential or session-refresh call, not a standalone auth check
- [ ] `NSFaceIDUsageDescription` set in `Info.plist`
- [ ] Biometric keypair enrolled on first login — public key registered with backend
- [ ] App locks after 30s in background (`AppState` listener configured)
- [ ] Biometric failure → PIN fallback → full logout after 3 failures
- [ ] Step-up biometric gate implemented for high-value actions (banking & payments)
- [ ] Biometric gate tested on physical device — Face ID, Touch ID, and Android fingerprint

## Session Management

- [ ] Session state machine implemented: `unauthenticated → authenticated → locked → refreshing → expired`
- [ ] Cold start always requires biometric gate — even if session token exists
- [ ] Logout clears: Auth0 refresh token, server-side session cookie, Keychain tokens, MMKV encryption key, cookie jar, Zustand/Redux store, navigation stack
- [ ] Navigation stack reset on logout — back-navigation to authenticated screens not possible
- [ ] Multi-device logout (`/auth/logout-all`) implemented and tested
- [ ] Forced logout on suspicious activity (backend-initiated session invalidation) handled client-side

## Backend Middleware

- [ ] JWT validation uses Auth0 JWKS endpoint — `RS256` algorithm enforced
- [ ] All sensitive routes require auth middleware before any handler logic
- [ ] Role-based access enforced from Auth0 custom claims — not from request body or query params
- [ ] Rate limiting on: login (10/15min), OTP send (5/15min), session-refresh (10/5min)
- [ ] Rate limiting uses Redis store — not in-memory (survives restarts, works across instances)
- [ ] Refresh token reuse detection implemented — family invalidation on replay
- [ ] Step-up endpoints validate challenge token expiry and single-use constraint
- [ ] Audit log records: login, logout, logout-all, OTP send, step-up initiation, reuse detection events

## Testing & Validation

### Token / Cookie Flow
- [ ] Concurrent request test: 3 simultaneous requests during refresh — Auth0 called exactly once
- [ ] Revoked refresh token test: verify navigation reset to Login
- [ ] Cookie flags verified on login response: `HttpOnly`, `Secure`, `SameSite=Strict`
- [ ] Session expiry verified: no request for 15 min → next request returns 401

### Biometric & Session
- [ ] Background lock test: app backgrounded > 30s → `locked` state on resume
- [ ] 3 biometric failures test → `expired` state → Keychain cleared
- [ ] Cold start test: existing session → biometric gate shown, not bypassed

### OTP & Step-Up
- [ ] OTP brute-force test: 6 rapid requests → 6th blocked with 429
- [ ] Step-up without proof: high-value action returns 403
- [ ] Expired challenge token: step-up attempt after 5 min returns 403

### Logout Completeness
- [ ] Post-logout: Keychain returns null for all auth entries
- [ ] Post-logout: cookie jar empty
- [ ] Post-logout: Auth0 refresh token rejected (revoked)
- [ ] Post-logout: back-navigation to authenticated screen not possible
- [ ] Post-logout: Zustand/Redux store cleared