# SICKSAINT Repository and Integration Audit

> **Superseding addendum (2026-11-24):** Firebase client auth and Laravel ID-token verification are in the current working tree, alongside the Next.js BFF. Read-only Firebase CLI/connector access now verifies project `sicksaint`, its Web App, default Standard Firestore database, active deny-all Firestore rules, and an empty deployed composite-index list. No production documents were read or written. Laravel service credentials/IAM and real auth remain unverified. The original local databases and subscriber JSON remain outside/absent from this export; Vercel integration remains unbuilt/unverified. See current [PROJECT_STATUS.md](PROJECT_STATUS.md) and [FINAL_QA_REPORT.md](FINAL_QA_REPORT.md). Older findings below are the prior baseline, not current claims.

**Audit date:** 2026-11-23  
**Scope:** current local monorepo candidate, including the original Next.js tree and Laravel backend candidate copied from `sicksaint-stage/backend`. No production service was changed.

## Repository inventory

- `frontend/`: existing Next.js 15.5.29 site and admin panel, with Next API routes, Prisma/SQLite, legacy JWT auth, and WordPress REST client.
- `backend/`: Laravel 12.73.0, PHP requirement `^8.2`, API routes/controllers, custom Firestore REST adapter, role middleware, token auth, migrations/schema assets, tests and Vercel FrankenPHP container definition.
- `database/`: separate Prisma workspace with a second schema; review role/schema mismatch before using its scripts. The current frontend's active Prisma datasource lives under `frontend/prisma/`.
- `docs/`, root manifests, Firestore policy candidate and emulator tests.

The backend is Laravel 12.73.0; this repository does not contain the requested Laravel 11 implementation. The backend candidate source was copied from the local `sicksaint-stage/backend` without copying `.env`, `vendor`, database files, service-account files or logs. No existing root `backend/` was overwritten.

## Auth and data sources

| Area | Current implementation | Verified state |
|---|---|---|
| Next public/admin app | Prisma/SQLite profiles and HS256 `ss_auth` cookie; API role checks reload the local user role. Admin UI is in Next.js. | Build succeeds; no real deployed session or durable Vercel DB verified. |
| Firebase browser SDK | Existing project Web App config loads Auth, Firestore, Storage. | SDK initialization and service endpoints probed read-only. Real user sign-in and Storage permission are untested. |
| Firebase Auth in Next | Optional sign-in button, ID token verified with Firebase Admin route, email matched to existing Prisma profile, local role minted into existing app cookie. | Code compiles; no Admin key or real Firebase user test. No Firebase UID mapping, account creation, signup, reset or migration. |
| Laravel Auth | Password hashes and opaque tokens stored in Firestore; role read from server-side user doc. | Feature tests pass with in-memory Firestore. Middleware does not verify Firebase ID tokens. Live server credentials absent. |
| WordPress | Next reads editorial content/categories through REST. | Production build logged failed REST fetches; live CMS not verified. |

The Next and Laravel auth/data systems are separate and conflict as parallel sources of truth. Do not claim the admin panel is backed by Laravel/Firestore or that auth has been unified.

## Firestore path inventory (Laravel source)

`users/{sha256(lowercase-trimmed-email)}`, `usernameIndex/{sha256(lowercase-username)}`, `accessTokens/{sha256(opaque-token)}`, `passwordResets/{sha256(lowercase-email)}`, `submissions/{uuid}`, `reviews/{uuid}`, `notifications/{uuid}`, `newsletterSubscribers/{sha256(normalized-email)}`, and optional `system/healthcheck`.

There were no direct Firestore data read/write call sites in the frontend. Public editorial reads are WordPress. Users, tokens, reset data, submissions, review details, notifications and subscriber addresses are private; admins perform privileged user/role listing, review/publishing, stats and export operations via application APIs. Laravel's server credential bypasses Firestore client rules; backend role and ownership checks must therefore remain mandatory.

See [docs/FIRESTORE_ACCESS_MODEL.md](docs/FIRESTORE_ACCESS_MODEL.md) for per-collection fields and query filters.

## Rules discrepancy

One unauthenticated production Web SDK `getDoc` request for a random nonexistent document succeeded on 2026-11-23, using configured project `sicksaint`, a fresh Node process and no emulator host. Request path prefix was `__sicksaint_readonly_<random-hex>/connection`; random suffix was not captured, so the exact full path cannot be reconstructed. No document data was returned. In this current audit, deployed rules were retrieved read-only and deny all client access, matching local rules. This confirms present rules state but cannot explain the historical earlier response because the request path suffix and ruleset state at that time are unavailable. No additional production document read occurred.

## Deployment findings

The root Vercel candidate defines Next and Laravel container services. Current official Vercel documentation describes multi-service deployments and container runtime, but these capabilities are beta and not verified in the user's account. The current rewrite sends only signed Laravel verification paths to Laravel; all remaining requests route to Next. `LARAVEL_API_URL` is declared but unused in frontend source. Next API routes still use Prisma/SQLite/local JSON, which are not durable on serverless storage. Container image build, CORS, Laravel health and runtime are unverified.

## QA summary

- Next.js production build: passed, 51 routes; WordPress fetch failures and image/font warnings remain.
- Laravel tests: 11 passed / 34 assertions using fake/in-memory Firestore.
- PHP lint: 71 files, 0 syntax failures.
- Laravel API routes: 25 registered.
- Firestore Emulator rules: 4/4 passed using a temporary checksum-verified Java 21 runtime.
- Production npm dependency audit: zero findings; full development audit earlier reported 13 moderate findings.

Detailed evidence and release blockers are in [PROJECT_STATUS.md](PROJECT_STATUS.md) and [FINAL_QA_REPORT.md](FINAL_QA_REPORT.md).
