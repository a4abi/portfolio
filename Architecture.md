# Architecture Deep Dive — Canada Tax Planner

**Companion doc to** `REVIEW.md`. Where the review flags problems, this doc explains **how the app is actually laid out**, layer by layer, with concrete file references.

---

## 1. High-Level Picture

```
┌───────────────────────────────────────────────────────────────────┐
│                          FLUTTER CLIENT                           │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐     │
│   │  Presentation layer (screens, Riverpod notifiers)       │     │
│   │  ─ rebuilds on state change, reads use cases only       │     │
│   └───────────────────┬─────────────────────────────────────┘     │
│                       │ calls                                     │
│   ┌───────────────────▼─────────────────────────────────────┐     │
│   │  Domain layer (entities, repo interfaces, use cases)    │     │
│   │  ─ pure Dart, no Flutter / Firebase imports             │     │
│   └───────────────────┬─────────────────────────────────────┘     │
│                       │ implemented by                            │
│   ┌───────────────────▼─────────────────────────────────────┐     │
│   │  Data layer (repo impls, models, data sources)          │     │
│   │  ─ talks to Firebase / mocks / HTTP functions           │     │
│   └───────────────────┬─────────────────────────────────────┘     │
│                       │                                           │
│   ┌───────────────────▼─────────────────────────────────────┐     │
│   │  Core + white_label + shared (cross-cutting)            │     │
│   │  ─ DI (GetIt), errors, constants, brand config, router  │     │
│   └─────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
                              │ HTTPS
┌───────────────────────────────────────────────────────────────────┐
│                       FIREBASE BACKEND                            │
│  Auth · Firestore · Storage                                       │
│  Cloud Functions (TS, Gen 2, northamerica-northeast2)             │
│     └─ extractReceiptFieldsHttp → Google Document AI              │
│     └─ deleteAccountHttp        → recursive user cleanup          │
└───────────────────────────────────────────────────────────────────┘
```

Three observations up front:

1. **Strict Clean Architecture**: domain has zero Flutter/Firebase imports. Verified in `lib/features/auth/domain/repos/auth_repo.dart` and `lib/core/domain/usecase.dart`.
2. **Dual DI**: `get_it` holds the object graph; Riverpod `Provider`s are thin "bridges" that read from `sl<T>()`. See `lib/features/auth/presentation/providers/auth_provider.dart:22-71`.
3. **Swappable backend**: `BackendType` enum (`firebase | rest | mock | aws`) drives which data source gets registered. Only Firebase and mock are wired today.

---

## 2. Directory Map (annotated)

```
lib/
├── main_comon.dart               # shared app bootstrap (typo preserved)
├── main_rushme.dart              # entry: brand = RushMe
├── main_brandb.dart              # entry: brand = BrandB
├── firebase_options.dart         # generated; holds Firebase web/android/ios keys
│
├── core/                         # CROSS-CUTTING, NO UI
│   ├── constants/                # enums, config values (fiscal years, MIME limits, CRA codes, subscription tiers, region/function names)
│   ├── di/
│   │   ├── injection_container.dart          # GetIt setup entry
│   │   └── injection_container.firebase.dart # Firebase-only registrations (isolated)
│   ├── domain/usecase.dart       # abstract UseCase<T, Params>
│   ├── errors/
│   │   ├── failures.dart         # sealed Failure hierarchy
│   │   └── result.dart           # typedef Result<T> = Either<Failure, T>
│   └── utils/                    # pure helpers (image compression, fiscal year math, email validation, MIME detection)
│
├── features/                     # FEATURE-FIRST, THREE LAYERS EACH
│   ├── auth/
│   │   ├── data/{datasources, models, repos}
│   │   ├── domain/{entities, repos, usecases}
│   │   └── presentation/{providers/, <screens>.dart}
│   ├── dashboard/                # receipts, income sources, profile, quotas
│   │   ├── data/{datasources, models, repos, catalog}
│   │   ├── domain/{entities, repos, usecases, <domain services>}
│   │   └── presentation/{helpers, providers, <screens>}
│   └── on_boarding/
│       ├── presentation/         # screens s12–s14
│       └── providers/            # Riverpod (code-gen via @riverpod)
│
├── shared/                       # REUSABLE ACROSS FEATURES
│   ├── router/app_router.dart    # GoRouter + redirect based on onboarding
│   └── widgets/                  # 60+ brand_* and home_* widgets
│
└── white_label/                  # BRAND-LEVEL CUSTOMIZATION (COMPOSITION, NOT INHERITANCE)
    ├── brand_config.dart         # immutable composite
    ├── brand_colors.dart, brand_typography.dart, brand_assets.dart, brand_strings.dart
    ├── feature_flags.dart        # hasTutorial, hasDocumentAi, hasAnalytics, …
    ├── backend_type.dart         # enum for swap-in backends
    ├── app_theme.dart            # Material theme derived from BrandConfig
    └── brands/
        ├── rushMefinancial_ux/rushme_brand_config.dart
        └── brandb/brandb_brand_config.dart
```

The three top-level groupings — `core`, `features`, `shared`, `white_label` — are the backbone. Each has a single job and (mostly) respects its own import boundary.

---

## 3. Layer-by-Layer Walk-through

### 3.1 Entry points (`lib/main_*.dart`)

There is **one shared bootstrap** and **one thin entry per brand**. `main_rushme.dart` is 8 lines:

```dart
import 'main_comon.dart';
import 'white_label/brands/rushMefinancial_ux/rushme_brand_config.dart';

void main() {
  mainCommon(rushMeFinancialUxBrandConfig);
}
```

`main_comon.dart` (sic) does the heavy lifting:

```dart
// lib/main_comon.dart:14
void mainCommon(BrandConfig config) async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  await initDependencies(config);                   // GetIt graph
  runApp(ProviderScope(
    overrides: [brandConfigProvider.overrideWithValue(config)],
    child: const AppRoot(),
  ));
}
```

**Why this matters**: the **brand is injected at the scope boundary**, not read from a singleton. Any widget can ask `ref.watch(brandConfigProvider)` and get the right colors/strings without global state.

Build with:
```
flutter run -t lib/main_rushme.dart     # ships Canada Tax Planner / RushMe
flutter run -t lib/main_brandb.dart     # ships the other brand
```

---

### 3.2 Core layer (`lib/core/`)

No UI. No feature knowledge. Only foundational primitives.

#### 3.2.1 Errors & Result type

```dart
// lib/core/errors/result.dart
typedef Result<T> = Either<Failure, T>;

// lib/core/errors/failures.dart
sealed class Failure { const Failure(this.message); final String message; }
class ServerFailure    extends Failure { const ServerFailure(super.m); }
class AuthFailure      extends Failure { const AuthFailure(super.m); }
class CacheFailure     extends Failure { … }
class NetworkFailure   extends Failure { … }
class ValidationFailure extends Failure { … }
class PIPEDAFailure    extends Failure { … }   // Canadian privacy domain vocabulary
```

Every use case returns `Future<Result<T>>`. Errors never throw across the domain boundary — they travel as `Left(Failure)`. Consumers pattern-match via `dartz`'s `.fold(onError, onSuccess)`.

#### 3.2.2 `UseCase<T, Params>` contract

```dart
// lib/core/domain/usecase.dart
abstract class UseCase<T, Params> {
  Future<Result<T>> call(Params params);
}
class NoParams { const NoParams(); }
```

Every use case in the app implements this. Calling a use case looks like `await signInGoogle(NoParams())`.

#### 3.2.3 Dependency Injection

Two files, purposefully split:

- `lib/core/di/injection_container.dart` — "backend-agnostic" — registers **repos**, **use cases**, **mock data sources**. This is the only file outside `data/` that imports concrete implementations, so the wiring is visible in one place.
- `lib/core/di/injection_container.firebase.dart` — **the only file in the app that imports `firebase_auth`, `cloud_firestore`, `firebase_storage`, `google_sign_in` SDKs** (line-3 comment says so explicitly). If you swap to another backend, you add a sibling `injection_container.aws.dart` and flip the enum.

Entry sequence:

```dart
// injection_container.dart:75
Future<void> initDependencies(BrandConfig config) async {
  _initReceiptQuotaCatalog();                     // subscription quota machinery
  switch (config.backendType) {
    BackendType.firebase: await initFirebaseBackend(sl, config);
    BackendType.mock:     _initMockRemoteDataSources();
    BackendType.rest/aws: throw UnsupportedError(...);
  }
  _initRepositories();                            // same across backends
  _initUseCases();                                // same across backends
  await _startReceiptQuotaListenIfReady();        // live Firestore listener
}
```

All registrations are guarded by `!sl.isRegistered<T>()` so re-initialization is idempotent — useful for hot reload and tests.

#### 3.2.4 Constants & utils

`core/constants/` holds domain vocabulary that would otherwise get stringly-typed: Canadian provinces, CRA expense codes, receipt state machine, subscription tiers, Firebase functions region/name. `core/utils/` holds pure Dart helpers (`fiscal_calendar_year.dart`, `receipt_image_compress.dart`, `email_validation.dart`). Nothing here owns state.

---

### 3.3 White-label layer (`lib/white_label/`)

This is the most distinctive piece of the codebase.

```dart
// lib/white_label/brand_config.dart
@immutable
class BrandConfig {
  final String brandId;
  final String appName;
  final BrandColors colors;
  final BrandTypography typography;
  final BrandAssets assets;
  final FeatureFlags featureFlags;
  final String apiBaseUrl;
  final BackendType backendType;
  final String privacyPolicyUrl;
  final String supportEmail;
  final BrandStrings brandStrings;
}
```

Design choices:

- **Composition, not inheritance.** `BrandConfig` aggregates five `Brand*` value objects plus feature flags. No `extends RushMeConfig`.
- **Feature flags live inside the brand**. `hasDocumentAi` flows from `BrandConfig.featureFlags.hasDocumentAi` → `initFirebaseBackend` → `ReceiptUploadRemoteDataSrcImpl` (`lib/core/di/injection_container.firebase.dart:60`) and gates whether the Cloud Function is even called (`receipt_upload_remote_data_src_impl.dart:66`). That's how a brand is "Pro-tier" without changing code paths elsewhere.
- **Brand is Riverpod-scoped.** `main_comon.dart:22` overrides `brandConfigProvider` on the `ProviderScope`. UI reads colors and strings via `ref.watch(brandConfigProvider).colors` / `.brandStrings` (see `dashboard_shell.dart:212-214`).
- **Brands can differ in backend**. `BrandConfig.backendType` lets RushMe run on Firebase while a white-labeled instance could run on REST/AWS.

The two brand configs (`rushme_brand_config.dart`, `brandb_brand_config.dart`) are essentially **data files** — the largest is ~315 lines of `const` string/color wiring.

---

### 3.4 Feature layer (`lib/features/<feature>/`)

Each feature is Clean Architecture's "vertical slice": **data + domain + presentation**, all under one folder. Dependencies flow **only** `presentation → domain ← data`. Domain never imports from data or presentation.

Take `auth` as the canonical example:

#### 3.4.1 Domain (`features/auth/domain/`)

- **Entities** (`entities/user.dart`) — pure Dart classes with `==`, `hashCode`, `copyWith`. Holds a clever "omit sentinel" pattern for optional-nullable fields (`UserEntity._omitSpouseEmail` lines 7-10, 64-72) so `copyWith(spouseEmail: null)` means "clear it" instead of "leave unchanged". This is the only way to express `null vs absent` without `Option<T>`. Keep that trick in mind; it's reused on `subscriptionEndDate`.
- **Repo interfaces** (`repos/auth_repo.dart`, `income_source_repo.dart`, `user_repo.dart`) — abstract. Each method returns `Future<Result<T>>`.
- **Use cases** (`usecases/`) — one file per action. A use case is *thin*: it takes a repo in the constructor and forwards the call:
  ```dart
  // domain/usecases/sign_in_google.dart
  class SignInGoogleUsecase implements UseCase<UserEntity, NoParams> {
    const SignInGoogleUsecase(this._repo);
    final AuthRepo _repo;
    @override
    Future<Result<UserEntity>> call(NoParams p) => _repo.signInWithGoogle();
  }
  ```
  The payoff is not hiding logic; it's **giving the presentation layer a stable, testable surface** independent of repo shape changes.

Notable auth use cases: `SignInGoogleUsecase`, `SignInAppleUsecase`, `SignOutUsecase`, `SaveRegistrationPersonalDetailsUsecase`, `FinishRegistrationUsecase`, `GetUserUsecase`, `WatchUserProfileUsecase`, `UpdateUserProfileUsecase`, `DeleteAccountUsecase`, plus income-source CRUD.

#### 3.4.2 Data (`features/auth/data/`)

- **Models** (`data/models/user_model.dart`) — extend the domain entity and add serialization. `UserModel.fromOAuthMap`, `UserModel.fromFirestoreMap`, `UserModel.fromEntity`. This is where `Timestamp → DateTime`, optional-string trimming, and default values live — keeping entity code serialization-free.
- **Data sources** (`data/datasources/`) — three flavors per noun:
  ```
  auth_remote_data_src.dart        # abstract interface
  auth_remote_data_src_impl.dart   # Firebase/Google impl
  auth_remote_data_src_mock.dart   # in-memory for tests / BackendType.mock
  ```
  The DI layer picks one based on `BackendType`. This is why `mock` is a first-class backend and not a test-only thing.
- **Repositories** (`data/repos/auth_repo_impl.dart`) — the **only place** raw exceptions are caught. They translate SDK errors (`FirebaseAuthException`, cancellation strings) into `AuthFailure` / `ServerFailure`. Example:
  ```dart
  // auth_repo_impl.dart:22-33
  on FirebaseAuthException catch (e) {
    return Left(AuthFailure(e.message ?? 'Authentication failed'));
  } catch (e) {
    if (msg.contains('GOOGLE_SIGN_IN_CANCELLED')) return Left(AuthFailure('Sign in cancelled'));
    if (msg.contains('GOOGLE_AUTH_MISSING_TOKEN')) return Left(AuthFailure('Could not obtain Google credentials'));
    return Left(AuthFailure(msg));
  }
  ```

#### 3.4.3 Presentation (`features/auth/presentation/`)

Two sub-concepts:

- **Screens** — `login_screen_s21.dart`, `registration_screen_s22.dart`, `registration_screen_s23.dart`, etc. File naming follows a design-system convention: `s21`, `s22`, `s23a`, `s23b` — these correspond to screens in the product spec. Easy to map to Figma/handoff docs.
- **Providers (Riverpod)** — `presentation/providers/auth_provider.dart` is the pattern for every feature:
  1. "DI bridge" providers: `signInGoogleUsecaseProvider = Provider((_) => sl<SignInGoogleUsecase>());`
  2. **Sealed-class state models**:
     ```dart
     sealed class SignInState {}
     final class SignInIdle     extends SignInState {}
     final class SignInSuccess  extends SignInState { final UserEntity user; final PostSignInRoute destination; }
     final class SignInError    extends SignInState { final String message; }
     ```
  3. **`AsyncNotifier` that composes use cases**:
     ```dart
     class SignInNotifier extends AsyncNotifier<SignInState> {
       FutureOr<SignInState> build() => const SignInIdle();
       Future<void> signInWithGoogle() async {
         state = const AsyncLoading();
         final signInResult = await ref.read(signInGoogleUsecaseProvider)(const NoParams());
         signInResult.fold(
           (failure) => state = AsyncData(SignInError(failure.message)),
           (user) async {
             final routeResult = await ref.read(resolvePostSignInRouteUsecaseProvider)(...);
             state = AsyncData(routeResult.fold(..., SignInSuccess.new));
           },
         );
       }
     }
     final signInNotifierProvider = AsyncNotifierProvider<SignInNotifier, SignInState>(SignInNotifier.new);
     ```

What to notice:
- **`AsyncLoading` wraps a sealed domain state** — you get *both* the async lifecycle (loading/data/error) *and* a typed application state (idle / success-with-destination / error-with-message) at the same time. No `isLoading && data != null` ambiguity.
- **The notifier orchestrates multiple use cases** (`signIn` then `resolvePostSignInRoute`). Orchestration belongs here, not in the use case; use cases stay single-purpose.

---

### 3.5 Dashboard feature — the interesting bits

`features/dashboard` is larger because it owns the receipts lifecycle, income sources, profile, subscription quotas, and Document AI integration. Same three-layer structure, plus two extras:

- **`domain/receipt_quota_catalog.dart`** — domain service for subscription tier limits, fed by `data/catalog/receipt_quota_catalog_impl.dart` and kept warm by `data/catalog/receipt_quotas_listen.dart` (attaches to Firestore on login, detaches on logout via `FirebaseAuth` hook in `injection_container.dart:96-98`).
- **`presentation/helpers/`** — view-model-ish pure functions (`receipt_list_format.dart`, `cra_expense_display.dart`, `receipt_form_apply_parsed_fields.dart`). They're stateless and used only by screens.

The most security-relevant file is `features/dashboard/data/datasources/receipt_upload_remote_data_src_impl.dart`:

- Builds the Cloud Function URL from `kFirebaseFunctionsRegion` + `DefaultFirebaseOptions.currentPlatform.projectId` (lines 35-45), with `kExtractReceiptFieldsHttpUrlOverride` escape hatch for staging/emulator.
- Forces an **ID token refresh** before calling the function (`user.getIdToken(true)` line 73), then sends `Authorization: Bearer <token>`. Matches what `extractReceiptFieldsHttp` in `functions/src/index.ts:167-186` validates.
- Writes receipt attachments to **`users/{uid}/receipts/{receiptId}/receipt`** — a single object name that overwrites on re-upload (see comment at line 124). This avoids duplicate filenames but means re-uploads are destructive.
- Gates the Document AI call on `_hasDocumentAi` (line 66), which is the `FeatureFlags.hasDocumentAi` value from the brand.

---

### 3.6 Shared layer (`lib/shared/`)

Just two things, but both important.

#### 3.6.1 Router

```dart
// lib/shared/router/app_router.dart:16
final routerProvider = Provider<GoRouter>((ref) {
  final notifier = _RouterNotifier(ref);   // listens to onboardingProvider
  final router = GoRouter(
    initialLocation: '/onboarding12',
    refreshListenable: notifier,
    redirect: (_, state) => notifier.redirect(state),
    routes: [ /* 7 screens */ ],
  );
  ref.onDispose(notifier.dispose);
  return router;
});
```

Key design decisions:

- **GoRouter is built *once* per ProviderScope**, not recreated on every widget rebuild. The comment at line 15 makes this explicit; losing this invariant tends to drop navigation state.
- The `_RouterNotifier` bridges a Riverpod provider (`onboardingProvider`) into GoRouter's `Listenable` expectation. On change, it calls `notifyListeners()` which re-runs `redirect`.
- **Redirect is a simple 2-rule gate**: if onboarding done and user is on an onboarding route → `/login`; if not done and user is off onboarding → `/onboarding12`.
- **Observation**: this router only covers onboarding + auth. The dashboard is entered as `/home → DashboardShell`, and `DashboardShell` handles all internal navigation via `IndexedStack`, `Navigator.push`, and drawer. Mixed strategies — declarative (`go_router`) for unauthenticated flow, imperative (`Navigator`) inside the authenticated shell.

#### 3.6.2 Widgets

Roughly 60 reusable widgets, mostly prefixed `brand_*` (theme-aware) or `home_*` (dashboard-specific). They accept `BrandColors`/`BrandStrings`/`BrandTypography` as parameters or read them via `ref.watch(brandConfigProvider)`. Examples: `brand_primary_button`, `brand_labeled_date_field`, `brand_modal_bottom_sheet`, `receipt_card_tile`. This is where the white-label system is actually *consumed*.

---

## 4. How a request flows (end-to-end)

To make the layers concrete, here's what happens when the user taps **"Scan receipt"** on the upload form:

```
UI                               Presentation            Domain              Data                  Firebase
──                               ────────────            ──────              ────                  ────────
upload_receipt_form_s35.dart ──▶ uploadReceiptProviders
                                   .extractNotifier
                                   .extractFrom(bytes) ─▶ ExtractReceiptFieldsUsecase
                                                            (domain/usecases) ────▶ ReceiptUploadRepo
                                                                                   (domain/repos)
                                                                                      │ impl
                                                                                      ▼
                                                                            ReceiptUploadRepoImpl
                                                                                      │
                                                                                      ▼
                                                                      ReceiptUploadRemoteDataSrcImpl
                                                                                      │
                                                    Authorization: Bearer <idToken>   │   HTTP POST
                                                    { mimeType, contentBase64 }       ▼
                                                                          extractReceiptFieldsHttp
                                                                          (functions/src/index.ts)
                                                                            │ verifyIdToken()
                                                                            │ MIME + size check
                                                                            ▼
                                                                        processReceiptWithDocumentAi
                                                                        (functions/src/document_ai_extract.ts)
                                                                            │ GoogleAuth ADC
                                                                            ▼
                                                                         Document AI REST
                                                                            (expense parser)
                                                                            │
                                                                            ▼
                                                                      {dateIso, netAmount, taxAmount,
                                                                       grossAmount, vendorName}
                                                                       ── whitelisted, PIPEDA-safe
                                                                            │
                                                                            ▼
                              ◀── Result<ReceiptParsedFields> ◀──── decode+validate ◀─── HTTP 200
```

Every arrow crosses exactly one layer boundary. The data source is the only thing that knows "Firebase Functions URL"; the repo is the only thing that knows "exceptions become Failures"; the use case is the only thing the notifier talks to; the UI only talks to notifiers.

---

## 5. Dependency & Import Rules (observed)

| Layer | May import from | Must not import from |
|-------|----------------|----------------------|
| `core/domain`, `features/*/domain` | `dartz`, other `domain/` | `flutter`, Firebase SDKs, `data/`, `presentation/` |
| `features/*/data` | `domain/`, `core/`, Firebase/HTTP SDKs | `presentation/` |
| `features/*/presentation` | `domain/` (usecases), `core/di` (via providers), `shared/`, `white_label/` | other features' `presentation/` or `data/` |
| `core/di/injection_container.firebase.dart` | Firebase SDKs | — (it's the *only* backend-SDK file outside `data/`) |
| `shared/widgets` | `white_label/`, Flutter | feature code (stays reusable) |
| `white_label/` | Flutter foundation only | feature/domain/data code |

These rules hold in the code I read. The one semi-violation: `features/dashboard/data/datasources/receipt_upload_remote_data_src_impl.dart` imports `firebase_options.dart` from lib root (line 11) to build the function URL. Tolerable — `firebase_options` is effectively configuration — but worth noting because it's the one thing outside the DI file that knows it's running on Firebase.

---

## 6. Cross-cutting Patterns

- **Error channel is `Result<T> = Either<Failure, T>`.** No exceptions escape the data layer. Use cases return `Future<Result<T>>`. Notifiers pattern-match with `.fold`.
- **State modelling is sealed-class + `AsyncNotifier`.** This combination gives you idle/success/error *and* loading, without nullable fields.
- **DI bridges** (Riverpod `Provider` → `sl<T>()`) keep use cases as a single resolution point while allowing Riverpod overrides in tests.
- **Feature flags gate entire code paths** at DI time (`hasDocumentAi`), not at widget build time. Brands that don't have a Pro feature don't pay its cost.
- **Firestore layout is user-scoped**: `users/{uid}/receipts/{receiptId}` (see `receipt_upload_remote_data_src_impl.dart:172-176`), `users/{uid}/incomeSources/...`. This maps 1:1 to the storage rule in `storage.rules`, which is why that rule is correct.
- **File naming mirrors the spec.** Screens are `*_s12.dart`, `*_s23a.dart`, etc. — one-to-one with the design spec's screen numbering. Easier review traceability than feature-name-only files.

---

## 7. Backend layer

Only two Cloud Functions, both in TypeScript, both Gen 2, both in `northamerica-northeast2`:

| Function | Purpose | Key guards |
|----------|---------|-----------|
| `deleteAccountHttp` (`functions/src/index.ts:30-124`) | Erase user data on account deletion | Verifies ID token, then `bucket.deleteFiles({ prefix: "users/${uid}/" })`, then `db.recursiveDelete(users/{uid})`, then `admin.auth().deleteUser(uid)`. Timeout 540s, memory 512MiB. |
| `extractReceiptFieldsHttp` (`functions/src/index.ts:141-265`) | OCR receipts via Document AI | Verifies ID token, validates MIME (PDF/JPEG/PNG/GIF/WebP/HEIC), enforces 7 MB max, base64-decodes, calls `processReceiptWithDocumentAi`. Response is a whitelist of 5 fields (`document_ai_extract.ts:14-20`) — explicit PIPEDA-minimizing step, not accidental. |

**Auth model.** Both functions check `Authorization: Bearer <idToken>` and call `admin.auth().verifyIdToken()`. The Flutter client forces a token refresh before each call (`receipt_upload_remote_data_src_impl.dart:73-77`), so handlers see fresh claims. Both are deployed with `invoker: "public"` — the review doc flags this; see `ARCHITECTURE_REVIEW.md` §H1.

**Document AI call** (`document_ai_extract.ts:208-254`) uses **Application Default Credentials** (`GoogleAuth`), meaning the function's runtime service account must have `roles/documentai.apiUser`. The processor URL is hardcoded to `projects/22341204437/.../processors/a8805783974f08ef:process`. That project number also appears in `firebase.json` and `firebase_options.dart`, so it's not new information leakage — just rigidity (review doc §M4).

---

## 8. Quick-reference: where to look when…

| Task | Start here |
|------|-----------|
| Add a new screen | `features/<feature>/presentation/<screen>_sXX.dart`, add a `GoRoute` in `shared/router/app_router.dart` (if public) or `Navigator.push` from `DashboardShell` (if authenticated). |
| Add a new domain action | Create use case in `features/<feature>/domain/usecases/`, register factory in `injection_container.dart:_initUseCases`. |
| Add a new data source | `features/<feature>/data/datasources/<noun>_remote_data_src.dart` (abstract), then `_impl.dart` and `_mock.dart`. Register both in `injection_container.firebase.dart` and `_initMockRemoteDataSources`. |
| Add a new brand | Create `white_label/brands/<brand>/<brand>_brand_config.dart`, add `main_<brand>.dart` that calls `mainCommon(brandConfig)`. No feature code changes required unless a new feature flag is needed. |
| Change Cloud Function contract | Edit `functions/src/index.ts`, rebuild (`npm --prefix functions run build`), update the Dart caller in `features/<feature>/data/datasources/*_impl.dart`. |
| Change Firestore schema | Edit the model in `features/<feature>/data/models/*_model.dart` (read+write), possibly the entity in `domain/entities/`. Remember there are currently no Firestore rules in the repo. |
| Toggle a brand capability | `white_label/feature_flags.dart` plus the consuming data-source constructor. |

---

## 9. Summary

The architecture is **classic Flutter Clean Architecture, executed faithfully**, with three distinguishing traits:

1. **White-label by composition** — `BrandConfig` as a Riverpod-scoped value, feature flags that gate DI, backend-type enum to swap data sources.
2. **Mock backend is a first-class citizen** — not a test helper, but a runtime backend you can boot into.
3. **Error handling is uniform** — `Result<T> = Either<Failure, T>`, enforced from the repo boundary to the notifier, with sealed `SignInState`-style application states sitting on top of `AsyncValue`.

The layering is real and load-bearing: you could delete `functions/` and the Firebase SDKs and, with a mock backend, the entire `domain`, `presentation`, and `shared` layers would still compile and run. That's the best signal that Clean Architecture is not decorative here.

The concerns flagged in `ARCHITECTURE_REVIEW.md` are almost entirely **infrastructure hygiene** (secrets in repo, missing Firestore rules, public invokers, test coverage, CI gaps) rather than structural flaws.
