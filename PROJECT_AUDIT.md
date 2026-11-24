# SICKSAINT Source and Integration Audit

> **Superseding addendum (2026-11-24):** Firebase CLI access is now verified for the existing `sicksaint` project. Read-only checks confirm its existing Web App, default Standard Firestore database, active deny-all client rules (matching this repo), and zero deployed composite indexes. No documents/users were enumerated and no production data/rules were modified. Laravel's service credential/IAM and live Auth remain unverified. The current working tree includes Firebase client sign-in/sign-up, a same-origin Laravel BFF, and Laravel protected middleware that verifies Firebase ID tokens and obtains roles from Firestore profiles. Older findings below are historical baseline where they conflict. See [PROJECT_STATUS.md](PROJECT_STATUS.md) and [FINAL_QA_REPORT.md](FINAL_QA_REPORT.md).

**Date:** 2026-11-23  
**Scope:** local source tree and non-destructive checks. See [FINAL_QA_REPORT.md](FINAL_QA_REPORT.md) for test evidence and [PROJECT_STATUS.md](PROJECT_STATUS.md) for release blockers.

## Repository layout and versions

- `frontend/`: existing Next.js 15.5.29 / React 18 / TypeScript application, public editorial pages, Next API routes, login/signup, contributor dashboard/editor and admin panel at `/admin`, `/admin/submissions`, `/admin/users`.
- `frontend/prisma/schema.prisma`: SQLite user, submission, review, notification and content-side models used by existing Next APIs.
- `database/`: a second Prisma workspace/schema with different models/roles; its seed and migrations must not be treated as the frontend's canonical schema without reconciliation.
- `backend/`: Laravel 12.73.0 (PHP `^8.2`) copied from local `sicksaint-stage/backend`; includes Firestore REST adapter, custom auth and role middleware, API routes/controllers and Vercel FrankenPHP Dockerfile. No `.env`, vendor code, local database or service-account file was copied.
- `docs/`, root package manifests, Firestore local rules and emulator tests.

This candidate does not contain a Laravel 11 source tree. It preserves the existing Next.js app and adds the available Laravel 12 candidate.

## Authentication and authorization findings

### Next.js

- Email/username-password registration and login use Prisma/SQLite, bcrypt, and an HS256 `ss_auth` HttpOnly cookie.
- `getAuthUser()` re-reads the profile and role from Prisma per API request; Admin APIs authorize using the current local database role rather than trusting a stale JWT role.
- Signup assigns `READER`. Admin UI is part of this app.
- Optional Firebase sign-in was added: verified Firebase email is mapped to an existing Prisma profile; Admin SDK verifies Firebase ID token and revocation; app role comes from Prisma. It does not mutate accounts or map UID. It needs Firebase Admin credentials; no real user test was available.
- Existing Firebase Auth signup, reset, account inventory, and account migration are not implemented.

### Laravel

- Laravel provides register/login/reset/verification/me/profile, submissions, reviews, notifications, newsletter and admin routes.
- Accounts use Firestore docs keyed by SHA-256 normalized email, `passwordHash`, role field, and hashed opaque-token docs. Middleware validates this Laravel token and loads the server-side role.
- It does not currently verify Firebase ID tokens. Its identity store/session are distinct from Next.js Prisma/JWT. Admin roles are server-side in each system, but they are not a unified source of truth.
- Feature tests passed using in-memory Firestore; the real Firebase service credentials are absent.

## Firestore path/access inventory

Laravel source uses `users/{sha256(email)}`, `usernameIndex/{sha256(username)}`, `accessTokens/{sha256(token)}`, `passwordResets/{sha256(email)}`, `submissions/{uuid}`, `reviews/{uuid}`, `notifications/{uuid}`, `newsletterSubscribers/{sha256(email)}`, and optional `system/healthcheck`. See [docs/FIRESTORE_ACCESS_MODEL.md](docs/FIRESTORE_ACCESS_MODEL.md) for field/query details and access classification.

The frontend has no direct Firestore data operations. Public articles/categories come through WordPress REST; profile/submission/admin APIs use Next/Prisma; the Laravel service independently uses Firestore. The candidate rules deny all browser access and are not published. Emulator tests deny unauthenticated, normal-user, and even admin-claim client access because admin operations must go through authorized servers.

One earlier unauthenticated production read of a random nonexistent Firestore document succeeded, conflicting with then-checked-in deny-all rules. Its path prefix was `__sicksaint_readonly_<random-hex>/connection`, but its random suffix was not saved. It returned no document contents. The request used Firebase Web SDK for `sicksaint`, in a fresh process with no emulator host. The user reports temporary deny-all rules have since been published; deployed rules were not independently fetched and no more production reads were made. The discrepancy remains unresolved.

## Editorial and storage findings

- Next.js server-side article/category paths use WordPress REST. The production build logged failed `/posts` and `/categories` requests; real public editorial content is unverified and may be empty.
- Some demo article records remain in an unused helper/fallback store. They are not evidence of live WordPress articles.
- `frontend/sicksaint-store.json` is tracked and contains 22 article records and 5 subscriber email records; values were not displayed. Do not package or push this file. Existing file was not changed. Fresh local storage initialization no longer seeds fabricated newsletter subscribers.
- Next APIs also use local SQLite and JSON for user/submission/newsletter operations. These are not durable shared storage for Vercel's serverless runtime.
- Secure upload/Storage workflows were not found. Firebase Storage client initialization exists only; no upload permissions or use path was tested.

## Vercel architecture

Root `vercel.json` declares Next and Laravel container services; `backend/Dockerfile.vercel` uses FrankenPHP. Official Vercel docs now describe Services/container support as beta. Access/enablement in this account, a Docker build, Laravel runtime and deployment have not been verified. The current route map sends signed Laravel email verification paths to Laravel and all other routes to Next. The frontend does not use the generated `LARAVEL_API_URL` binding. Therefore the current public/admin APIs are not backed by Laravel.

## Executed checks

- Next.js production build: passed with 51 routes and type validation; image/font warnings and WordPress fetch failures remain.
- Laravel Composer manifest: valid.
- Laravel tests: 11 passed / 34 assertions against fake/in-memory Firestore.
- PHP syntax: 71 source files passed, 0 failures.
- API route registration: 25 routes listed.
- Firestore Emulator: four deny-all tests passed earlier. Re-run currently requires Java 21+ on PATH; subsequent attempt was unable to start Java.
- Production dependency audit: zero vulnerabilities. Full dev dependency audit earlier showed 13 moderate findings.

## Release blockers

See [PROJECT_STATUS.md](PROJECT_STATUS.md). In brief: missing Firebase Admin credentials/account inventory, separate auth/data systems and no Firebase ID-token auth in Laravel, Next API not wired to Laravel, local SQLite/JSON persistence, unconfigured WordPress, unverified Vercel Services/container capability, and tracked local subscriber data that must be excluded from any release copy.
