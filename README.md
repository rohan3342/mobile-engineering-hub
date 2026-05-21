# Mobile Engineering Hub

Welcome to my Mobile Engineering Hub!

This repository is my personal knowledge base for mobile application development — technical deep dives, architectural notes, implementation guides, and code patterns covering the React Native ecosystem.

## What's Inside

### App Security
Production-grade security patterns for React Native apps, with a focus on banking, payments, and regulated industries.

- [Defense-in-Depth Implementation Guide](docs/security/01-rn-security-defense-in-depth.md) — JailMonkey, freeRASP, Firebase App Check, SSL Pinning, Encrypted Storage, Sardine SDK, and Secret Management
- [Pre-Launch Security Checklist](docs/security/02-security-checklist.md) — a gate-ready checklist covering every layer before shipping to production

### Authentication
End-to-end authentication architecture for React Native — from Auth0 integration and token strategy to biometric gates, session management, and backend enforcement.

- [Auth Architecture Guide](docs/auth/02-rn-auth-architecture.md) — Auth0 (Pattern A & B), JWT Bearer tokens, HttpOnly Cookie sessions, Social login, OTP/Passwordless, Biometric re-auth, Session state machine, and Backend reference (Node/Express + Spring Boot)
- [Pre-Launch Auth Checklist](docs/auth/03-auth-checklist.md) — a gate-ready checklist covering every auth layer before shipping to production

---

## Coming Soon

- **CI/CD Pipeline** — Fastlane + GitHub Actions for React Native (Android & iOS)
- **Offline-First Data Sync** — conflict resolution, optimistic updates, background sync
- **Performance & Bundle Optimization** — Hermes, Metro config, lazy loading, startup time

---

## Repository Structure

```text
mobile-engineering-hub/
├── README.md                  # You are here
├── SUMMARY.md                 # GitBook sidebar navigation
├── .gitbook.yaml              # GitBook configuration
├── assets/                    # Diagrams, architecture charts, images
└── docs/
    ├── security/
    │   ├── 01-rn-security-defense-in-depth.md
    │   └── 02-security-checklist.md
    └── auth/
        ├── 02-rn-auth-architecture.md
        └── 03-auth-checklist.md
```