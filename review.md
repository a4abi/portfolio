# Architecture Review — Canada Tax Planner (Flutter)

**Date:** 2026-05-05
**Scope:** `/Users/abshankar/git/test/taxplanner`
**Stack:** Flutter/Dart client (multi-platform) + Firebase backend (Firestore, Storage, Auth, Cloud Functions in TypeScript) + Google Document AI.

---

## 1. Architecture Overview

### 1.1 Pattern
The `lib/` tree follows **Clean Architecture with feature-first organization**:

```
lib/
├── core/                 # constants, DI, shared domain primitives, errors, utils
├── features/
│   ├── auth/             # data / domain / presentation
│   ├── dashboard/        # data / domain / presentation (receipts)
│   └── on_boarding/
├── shared/               # go_router, common widgets
└── white_label/          # per-brand configs (RushMe, BrandB) + BackendType abstraction
```

Each feature splits into `data` (datasources, models, repo impls), `domain` (entities, repo interfaces, usecases), and `presentation` (screens, Riverpod providers). Domain layer has no Flutter/Firebase imports — clean.

### 1.2 State & DI
- **State:** `flutter_riverpod ^3.3.1` with `AsyncNotifier` and sealed-class states (e.g. `lib/features/auth/presentation/providers/auth_provider.dart`).
- **DI:** `get_it ^9.2.1` service locator, bridged into Riverpod via provider overrides.
- **Backend abstraction:** `lib/white_label/backend_type.dart` declares `firebase | mock | rest | aws`, enabling swap-in data sources (currently only firebase + mock wired).
- **Branding:** ProviderScope overrides inject brand config at app start; three entry points (`main_comon.dart`, `main_rushme.dart`, `main_brandb.dart`).

### 1.3 Key Dependencies
Firebase suite (`firebase_core`, `cloud_firestore`, `firebase_storage`, `firebase_auth`), `go_router ^17`, `dartz` (Either error channel), `freezed`, `equatable`, `google_sign_in`, `sign_in_with_apple`, `crypto`, `pdfx`, `image_picker`, `file_picker`, `flutter_image_compress`.

### 1.4 Backend
- **Cloud Functions (TS, Gen 2, region `northamerica-northeast2`)** in `functions/src/index.ts`:
  - `deleteAccountHttp` — verifies Firebase ID token, recursively deletes user Firestore data (540s timeout, 512MB).
  - `extractReceiptFieldsHttp` — verifies ID token, proxies image/PDF to Google Document AI, returns whitelist of fields (`dateIso`, `netAmount`, `taxAmount`, `grossAmount`, `vendorName`).
- **Storage rules** (`storage.rules`): scoped to `/users/{uid}/receipts/**`, requires `request.auth.uid == userId`. Correct.
- **Firestore rules:** **NOT PRESENT in repo.** `firebase.json` does not reference a `firestore.rules` file.

### 1.5 Platforms
Android, iOS, Web, Windows directories all present. CI (`.gitlab-ci.yml`) only builds Android release; every other stage is commented out.

---

## 2. Findings by Priority

### 2.1 CRITICAL

#### C1. Android signing keystore committed to repo
- **File:** `keystore_base64.txt` (3,512 bytes, base64 PKCS12 keystore)
- **Why it matters:** anyone with repo access can sign APKs that Android will accept as genuine updates of the shipped app. If this is the production upload key, the Play Store signing identity is compromised.
- **Not in `.gitignore`.** It is tracked.
- **Action:**
  1. Treat the key as compromised — rotate the upload key via Play Console (Play App Signing allows upload-key reset).
  2. `git rm` the file, purge it from history (`git filter-repo` / BFG), force-push, rotate.
  3. Store the new keystore in CI secret storage only (GitLab CI variables, file type, masked).
- **Alternative:** if this is only a dev/debug keystore, document that explicitly in the file name and README, and still remove from history — a casual reader cannot tell dev from prod.

#### C2. No Firestore security rules in repo
- **Evidence:** no `firestore.rules` file; `firebase.json` lacks a `"firestore"` block.
- **Why it matters:** rules may exist only in the Firebase console (not version-controlled, not peer-reviewed), or they may be the default locked-down rules that silently block the app. Either way, the security posture of the primary data store is invisible from the repo.
- **Action:** add `firestore.rules` and register it in `firebase.json`:
  ```json
  "firestore": { "rules": "firestore.rules", "indexes": "firestore.indexes.json" }
  ```
  Start from a least-privilege baseline, e.g.:
  ```
  match /users/{uid}/{document=**} {
    allow read, write: if request.auth != null && request.auth.uid == uid;
  }
  ```
  Then open up specific collections as features demand.
- **Alternative:** if the team prefers callable functions over direct Firestore access from the client, lock rules down entirely (`allow read, write: if false;`) and route all writes through Cloud Functions. Costs more function invocations but gives you a single choke point for validation and audit.

### 2.2 HIGH

#### H1. Cloud Functions deployed with `invoker: "public"`
- **File:** `functions/src/index.ts` (both functions).
- **Why it matters:** anyone on the internet can hit the endpoint. ID-token verification inside the handler prevents data access, but the endpoint itself can still be spammed — each request costs a cold start and compute. Document AI is pay-per-page.
- **Action:** add **Firebase App Check** (`X-Firebase-AppCheck` header is already listed in CORS, but not enforced in the handler). Enforce with `firebase-admin/app-check`:
  ```ts
  const appCheckToken = req.header("X-Firebase-AppCheck");
  await getAppCheck().verifyToken(appCheckToken); // throws if invalid
  ```
  This restricts callers to your real app builds.
- **Alternative:** switch to `onCall` (HTTPS callable) functions — they get built-in auth context and App Check enforcement via a single flag. Requires a small client change (use `cloud_functions` package) but removes hand-rolled token-verification code.

#### H2. No rate limiting on expensive endpoints
- **Why it matters:** `extractReceiptFieldsHttp` calls Document AI (billed per page). An authenticated attacker can drain the budget.
- **Action:** per-user quota in Firestore (counter doc with server-timestamp bucket) checked at handler start; reject with 429 when exceeded. Keep limits generous (e.g. 50 receipts/day/user).
- **Alternative:** Cloud Armor rate-limiting rule in front of the function, or move to Firebase Extensions' rate-limited Cloud Functions.

#### H3. `cors: true` allows any origin
- **File:** `functions/src/index.ts` line ~40-46.
- **Why it matters:** web clients from any domain can trigger the functions. Combined with H1, this widens the abuse surface.
- **Action:** pin CORS to the deployed web origins:
  ```ts
  cors: ["https://canada-tax-planner.web.app", "https://<custom domain>"]
  ```
- **Alternative:** if no web client exists yet, set `cors: false` and add it back when the web build ships.

#### H4. Firestore API keys and GCP project numerics in repo
- **Files:** `lib/firebase_options.dart`, `functions/src/document_ai_extract.ts` (Document AI URL with project `22341204437`).
- **Why it matters:** Firebase "API keys" are public-by-design (they identify the project, not authenticate it), so this is **not a credential leak**. The risk is that without Firestore rules (C2) or App Check (H1), these identifiers are enough to probe the project.
- **Action:** accept that these are public, and compensate with strong rules + App Check.
- **Alternative:** restrict each API key in the GCP Console by Android package / iOS bundle / HTTP referrer so only your app binaries can use them.

### 2.3 MEDIUM

#### M1. Test coverage is essentially zero
- **File:** `test/widget_test.dart` — single smoke test.
- **Why it matters:** Clean Architecture's main payoff is testability; nothing is being tested. No tests for usecases, repositories, Cloud Functions, or form validation.
- **Action:** prioritize, in order:
  1. Domain usecase tests (pure Dart, fast, no Firebase).
  2. Repository tests with `mocktail`, against the existing mock datasources.
  3. TypeScript function tests with `firebase-functions-test` for ID-token verification and Document AI error paths.
  4. Widget tests per feature entry screen.
- **Alternative:** if the team won't invest in broad coverage, require tests only for the billing-sensitive paths (auth, receipt extraction, account deletion) and skip the rest.

#### M2. CI pipeline is mostly commented out
- **File:** `.gitlab-ci.yml` — only Android build runs; test, iOS, deploy, security stages disabled.
- **Action:** re-enable at minimum `flutter analyze` and `flutter test` on every MR. Add `npm --prefix functions run build && npm --prefix functions test`. Fail the build on any analyzer warning (matches the strict `analysis_options.yaml`).
- **Alternative:** start with a single "verify" job (analyze + test, both platforms) and expand once it's green; commented-out YAML usually rots.

#### M3. `.gitignore` does not cover secrets or generated files
- Missing: `*.env`, `.env.*`, `*.keystore`, `*.jks`, `*.p12`, `keystore*.txt`, `**/google-services.json`, `**/GoogleService-Info.plist`.
- **Action:** add those patterns. Add a `pre-commit` hook (gitleaks or `detect-secrets`) to block accidental re-introduction.
- **Alternative:** enable GitLab's built-in secret detection in CI — catches leaks post-push but requires no local tooling.

#### M4. Hardcoded Document AI processor URL
- **File:** `functions/src/document_ai_extract.ts:11-12`.
- **Why it matters:** switching processors (e.g. to a newer expense parser) requires a code change and redeploy; also makes staging vs prod harder.
- **Action:** move to environment config:
  ```ts
  const url = process.env.DOCUMENT_AI_URL!;
  ```
  Set via `firebase functions:config:set` or Gen-2 env vars.

### 2.4 LOW

#### L1. README is 9 lines
- No setup instructions, no description of the white-label mechanism or of the three `main_*.dart` entrypoints. New engineers will be lost.
- **Action:** expand to cover: prerequisites, `flutter pub get` + codegen, how to run each brand (`flutter run -t lib/main_rushme.dart`), Firebase emulator usage, deploy.

#### L2. Lingering TODO in auth flow
- `lib/features/auth/presentation/registration_screen_s23a.dart` — date picker `firstDate: DateTime(1900)` with TODO to narrow the range. Minor UX issue.

#### L3. `src/**` excluded from deploy but TypeScript already compiles to `lib/`
- `firebase.json` ignores `src/**`, which is fine. Confirm the compiled output directory matches `main` in `functions/package.json` — if not, deploy ships nothing.

---

## 3. Positive Observations

- **Clean layering is real, not decorative.** Domain has no Firebase imports; repo interfaces live in domain with implementations in data. Swapping backends (already stubbed via `BackendType`) is feasible.
- **Auth flows correctly verify Firebase ID tokens server-side** before doing anything sensitive (`functions/src/index.ts`).
- **Document AI response is whitelisted** to five fields — good PIPEDA posture, avoids accidentally persisting OCR'd PII.
- **Storage rules are correct** — UID-scoped, write-gated.
- **`analysis_options.yaml` is strict** (`prefer_const_constructors`, `avoid_positional_boolean_parameters`, `matching_super_parameters`). Combined with `flutter_lints ^6`, code quality floor is high.
- **White-label architecture** is unusually well thought out for a project this size — brand config is data, not inheritance, and injected via Riverpod overrides.
- **Sealed-class state modelling** in Riverpod notifiers prevents the usual `isLoading && data != null` ambiguity.

---

## 4. Prioritized Remediation Plan

| # | Severity | Action | Effort |
|---|----------|--------|--------|
| 1 | CRITICAL | Rotate keystore, purge `keystore_base64.txt` from git history | 0.5d |
| 2 | CRITICAL | Author + deploy `firestore.rules`, register in `firebase.json` | 1d |
| 3 | HIGH | Enforce Firebase App Check in both Cloud Functions | 0.5d |
| 4 | HIGH | Pin `cors` to known origins | 0.25d |
| 5 | HIGH | Add per-user daily quota to `extractReceiptFieldsHttp` | 0.5d |
| 6 | MEDIUM | Re-enable `flutter analyze` + `flutter test` in CI | 0.25d |
| 7 | MEDIUM | Harden `.gitignore` + add gitleaks pre-commit | 0.25d |
| 8 | MEDIUM | Seed usecase + repository unit tests (auth, receipts) | 2–3d |
| 9 | MEDIUM | Move Document AI URL to env config | 0.25d |
| 10 | LOW | Expand README (entrypoints, codegen, emulators) | 0.5d |

---

## 5. TL;DR

Architecture is solid — feature-first Clean Architecture, Riverpod + GetIt, good separation of brand config, sensible Cloud Functions design. The code will scale. **The security posture around secrets and access control will not**: a signing keystore is committed, Firestore rules aren't version-controlled, and the Cloud Functions are publicly invokable without App Check or rate limits. Fix items 1–5 before the next release; items 6–10 can ride along over the following sprints.
