# Flutter Reusable Packages

> A catalog of reusable Flutter packages designed as **git submodules** for modular architecture apps.
> Each package is self-contained, independently testable, and follows clean architecture principles.

---

## Architecture Overview

Packages are organized into **four layers**, where dependencies flow **downward only**.

```
┌─────────────────────────────────────────────┐
│            Feature Packages                 │  ← Depends on Core + Domain + Data
│  (auth, onboarding, settings, chat, etc.)   │
├─────────────────────────────────────────────┤
│            Data Packages                    │  ← Depends on Core + Domain
│  (network, storage, analytics, etc.)        │
├─────────────────────────────────────────────┤
│            Domain Packages                  │  ← Depends on Core only
│  (models, use_cases, repositories)          │
├─────────────────────────────────────────────┤
│            Core Packages                    │  ← No internal dependencies
│  (theme, utils, localization, navigation)   │
└─────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Dependency Rule**: A package may only depend on packages in the same layer or layers below it. Feature packages must **never** depend on other feature packages directly—use shared domain or core packages instead.

---

## 1. Core Packages

Foundation packages with **zero internal dependencies**. Every other package may depend on these.

---

### 1.1 `core_ui` — Design System & Theme

A single source of truth for all visual styling across apps.

| Aspect               | Detail                                                                                                                                                                             |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**          | App-wide theme, typography, colors, spacing tokens, reusable widgets                                                                                                               |
| **Design Pattern**   | **Token-based Design System** — expose primitives (`AppColors`, `AppTypography`, `AppSpacing`) and semantic tokens (`AppTheme`)                                                    |
| **Key Exports**      | `AppTheme`, `AppColors`, `AppTypography`, `AppSpacing`, `AppShadows`, `AppRadius`                                                                                                  |
| **Reusable Widgets** | `AppButton`, `AppTextField`, `AppCard`, `AppDialog`, `AppBottomSheet`, `AppSnackbar`, `AppLoadingIndicator`, `AppEmptyState`, `AppErrorWidget`, `AppAvatar`, `AppBadge`, `AppChip` |
| **Testing Focus**    | Widget tests for every reusable widget; golden tests for visual regression                                                                                                         |

```
core_ui/
├── lib/
│   ├── src/
│   │   ├── tokens/         # colors, typography, spacing, shadows, radius
│   │   ├── theme/          # AppTheme (light/dark), ThemeExtensions
│   │   └── widgets/        # all reusable widgets
│   └── core_ui.dart        # barrel export
├── test/
│   ├── widgets/            # widget tests
│   └── goldens/            # golden image tests
└── pubspec.yaml
```

---

### 1.2 `core_utils` — Shared Utilities

Pure Dart utilities with no Flutter dependency where possible.

| Aspect             | Detail                                                                                                                                                                                                                        |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Extensions, formatters, validators, constants, helpers                                                                                                                                                                        |
| **Key Exports**    | `StringExtensions`, `DateTimeExtensions`, `NumExtensions`, `Validators` (email, phone, password), `Formatters` (currency, date, number), `Result<T>` type (for functional error handling), `Debouncer`, `Throttler`, `Logger` |
| **Design Pattern** | **Result Pattern** — `Result<T>` with `Success<T>` and `Failure` subtypes to replace `try/catch` in business logic                                                                                                            |
| **Testing Focus**  | 100% unit test coverage; these are pure functions                                                                                                                                                                             |

```
core_utils/
├── lib/
│   ├── src/
│   │   ├── extensions/
│   │   ├── validators/
│   │   ├── formatters/
│   │   ├── result/          # Result<T>, Success<T>, Failure
│   │   ├── helpers/         # Debouncer, Throttler, Logger
│   │   └── constants/
│   └── core_utils.dart
├── test/
│   ├── extensions/
│   ├── validators/
│   ├── formatters/
│   └── result/
└── pubspec.yaml
```

---

### 1.3 `core_localization` — Localization Infrastructure

Shared localization setup and utilities.

| Aspect             | Detail                                                                                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Purpose**        | Base localization setup, locale management, common translated strings                                                                                  |
| **Key Exports**    | `LocalizationConfig`, `LocaleProvider`, `AppLocalizations` base class, common strings (OK, Cancel, Error, etc.), RTL helpers                           |
| **Design Pattern** | **Delegate Pattern** — each feature package provides its own `LocalizationsDelegate`, this package provides the base infrastructure and shared strings |
| **Testing Focus**  | Unit tests for locale switching, RTL detection, string interpolation                                                                                   |

---

### 1.4 `core_navigation` — Routing Infrastructure

Abstracted navigation layer built on top of `go_router`.

| Aspect             | Detail                                                                                                                                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Centralized route definitions, deep linking support, navigation guards, route-aware analytics                                               |
| **Key Exports**    | `AppRouter` base, `RouteDefinition`, `NavigationGuard` interface, `RouteObserver`, typed route parameters                                   |
| **Design Pattern** | **Strategy Pattern** for navigation guards — define `NavigationGuard` interface, implement `AuthGuard`, `OnboardingGuard`, `RoleGuard` etc. |
| **Dependencies**   | `go_router`                                                                                                                                 |
| **Testing Focus**  | Unit tests for each guard; integration tests for route transitions                                                                          |

```dart
// NavigationGuard interface (Strategy Pattern)
abstract class NavigationGuard {
  /// Returns redirect path if guard fails, null if pass
  String? evaluate(GoRouterState state, AuthStatus authStatus);
}

class AuthGuard implements NavigationGuard { ... }
class OnboardingGuard implements NavigationGuard { ... }
```

---

## 2. Data Packages

Handle external I/O: networking, local storage, file handling, analytics.

---

### 2.1 `data_network` — HTTP & API Client

Centralized, configurable HTTP client.

| Aspect             | Detail                                                                                                                                                                     |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | REST API client, request/response interceptors, error mapping, retry logic                                                                                                 |
| **Key Exports**    | `ApiClient`, `ApiException`, `ApiResponse<T>`, `Interceptor` interface, `RetryPolicy`                                                                                      |
| **Design Pattern** | **Interceptor/Chain of Responsibility Pattern** — requests pass through a chain of interceptors (auth token injection, logging, caching, retry) before reaching the server |
| **Dependencies**   | `dio`, `core_utils` (for `Result<T>`)                                                                                                                                      |
| **Testing Focus**  | Unit tests with mocked HTTP; integration tests with mock server                                                                                                            |

```dart
// Interceptor Chain of Responsibility
class ApiClient {
  final List<ApiInterceptor> interceptors;

  Future<Result<T>> request<T>(ApiRequest request) async {
    var processedRequest = request;
    for (final interceptor in interceptors) {
      processedRequest = await interceptor.onRequest(processedRequest);
    }
    // ... execute and run response through interceptors
  }
}
```

---

### 2.2 `data_local_storage` — Persistent Storage

Unified local storage with type safety.

| Aspect             | Detail                                                                                                                                                                            |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Key-value storage (preferences), secure storage (tokens/secrets), structured DB (offline cache)                                                                                   |
| **Key Exports**    | `StorageService` interface, `PreferencesStorage`, `SecureStorage`, `DatabaseStorage`, `CachePolicy`                                                                               |
| **Design Pattern** | **Repository Pattern** with **Adapter Pattern** — `StorageService` is the abstract interface; adapters wrap `shared_preferences`, `flutter_secure_storage`, and `drift`/`sqflite` |
| **Dependencies**   | `shared_preferences`, `flutter_secure_storage`, `drift` (optional)                                                                                                                |
| **Testing Focus**  | Unit tests with in-memory implementations; integration tests for each adapter                                                                                                     |

---

### 2.3 `data_analytics` — Analytics & Crash Reporting

Unified analytics interface supporting multiple providers.

| Aspect             | Detail                                                                                                                                                            |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Event tracking, screen tracking, user properties, crash reporting                                                                                                 |
| **Key Exports**    | `AnalyticsService` interface, `AnalyticsEvent`, `FirebaseAnalyticsAdapter`, `MixpanelAdapter`, `CrashlyticsAdapter`, `CompositeAnalyticsService`                  |
| **Design Pattern** | **Adapter Pattern** + **Composite Pattern** — each analytics provider has its own adapter; `CompositeAnalyticsService` fans out events to all registered adapters |
| **Dependencies**   | Provider SDKs (Firebase, Mixpanel, etc. — all optional)                                                                                                           |
| **Testing Focus**  | Unit tests verifying event dispatch to each adapter; mock adapters for testing                                                                                    |

```dart
// Composite Pattern
class CompositeAnalyticsService implements AnalyticsService {
  final List<AnalyticsService> _services;

  @override
  Future<void> logEvent(AnalyticsEvent event) async {
    await Future.wait(_services.map((s) => s.logEvent(event)));
  }
}
```

---

### 2.4 `data_file_handling` — File Uploads, Downloads & Image Processing

| Aspect             | Detail                                                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | File picking, image compression, upload/download with progress, file caching                                                          |
| **Key Exports**    | `FileService`, `ImageCompressor`, `UploadTask`, `DownloadTask`, `FileCache`                                                           |
| **Design Pattern** | **Command Pattern** — upload and download tasks are encapsulated as commands with progress tracking, pause/resume/cancel capabilities |
| **Dependencies**   | `image_picker`, `image`, `path_provider`                                                                                              |
| **Testing Focus**  | Unit tests for compression logic; integration tests for upload/download flows                                                         |

---

### 2.5 `data_connectivity` — Network Connectivity Monitoring

| Aspect             | Detail                                                                                        |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Purpose**        | Real-time connectivity status, offline queue, auto-retry on reconnection                      |
| **Key Exports**    | `ConnectivityService`, `ConnectivityStatus` enum, `OfflineQueue`                              |
| **Design Pattern** | **Observer Pattern** — broadcast connectivity changes via streams; consumers react to changes |
| **Dependencies**   | `connectivity_plus`                                                                           |
| **Testing Focus**  | Unit tests with mock connectivity; verify offline queue behavior                              |

---

## 3. Domain Packages

Business logic layer. Pure Dart, no Flutter imports.

---

### 3.1 `domain_models` — Shared Data Models

| Aspect             | Detail                                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Shared entities, value objects, enums used across features                                                                |
| **Key Exports**    | `User`, `AppConfig`, `PaginatedList<T>`, `GeoLocation`, `DateRange`, `Money`, common enums                                |
| **Design Pattern** | **Value Object Pattern** — models are immutable with `copyWith`, `==`, `hashCode`; use `freezed` or manual implementation |
| **Dependencies**   | `freezed_annotation`, `json_annotation` (code gen only)                                                                   |
| **Testing Focus**  | Unit tests for serialization/deserialization, equality, copyWith                                                          |

---

### 3.2 `domain_auth` — Authentication Domain Logic

| Aspect             | Detail                                                                                             |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| **Purpose**        | Auth state management, token lifecycle, session handling                                           |
| **Key Exports**    | `AuthRepository` interface, `AuthStatus` enum, `AuthCredentials`, `AuthToken`, `AuthException`     |
| **Design Pattern** | **Repository Pattern** — `AuthRepository` defines the contract; implementations live in data layer |
| **Testing Focus**  | Unit tests for auth state transitions, token refresh logic                                         |

---

## 4. Feature Packages

Complete, self-contained features. Each owns its UI, BLoC, and routing.

> [!NOTE]
> Every feature package follows the same internal structure using the **BLoC Pattern** for state management.

### Feature Package Internal Structure

```
feature_xxx/
├── lib/
│   ├── src/
│   │   ├── bloc/            # Feature BLoC(s), Events, States
│   │   ├── models/          # Feature-specific models (not shared)
│   │   ├── repositories/    # Feature-specific repository implementations
│   │   ├── services/        # Feature-specific business logic services
│   │   ├── screens/         # Full-page UI widgets
│   │   ├── widgets/         # Feature-scoped reusable widgets
│   │   └── l10n/            # Feature-specific localization ARB files
│   └── feature_xxx.dart     # Barrel export
├── test/
│   ├── bloc/                # BLoC tests with mocktail
│   ├── repositories/
│   ├── screens/             # Widget tests
│   └── integration/         # Integration tests
└── pubspec.yaml
```

---

### 4.1 `feature_auth` — Authentication UI & Flow

| Aspect            | Detail                                                                                                                                 |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**       | Sign in, sign up, forgot password, email verification, social auth screens                                                             |
| **Key Screens**   | `SignInScreen`, `SignUpScreen`, `ForgotPasswordScreen`, `EmailVerificationScreen`                                                      |
| **BLoC**          | `AuthBloc` — handles `SignInRequested`, `SignUpRequested`, `ForgotPasswordRequested`, `SignOutRequested`, `EmailVerificationRequested` |
| **Dependencies**  | `core_ui`, `core_utils`, `core_localization`, `core_navigation`, `domain_auth`, `data_local_storage`                                   |
| **Testing Focus** | BLoC tests for all auth flows; widget tests for form validation; integration tests for full auth journey                               |

---

### 4.2 `feature_onboarding` — First-Time User Experience

| Aspect             | Detail                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------ |
| **Purpose**        | App introduction slides, permissions requests, profile setup wizard                        |
| **Key Screens**    | `OnboardingScreen` (slides), `PermissionsScreen`, `ProfileSetupScreen`                     |
| **BLoC**           | `OnboardingBloc` — tracks completion state, persists "seen" flag                           |
| **Design Pattern** | **State Machine Pattern** — onboarding progresses through defined steps; cannot skip ahead |
| **Dependencies**   | `core_ui`, `core_localization`, `data_local_storage`                                       |
| **Testing Focus**  | BLoC tests for step transitions; widget tests for each slide                               |

---

### 4.3 `feature_settings` — App Settings & Preferences

| Aspect            | Detail                                                                                          |
| ----------------- | ----------------------------------------------------------------------------------------------- |
| **Purpose**       | Theme switching, language selection, notification preferences, account management               |
| **Key Screens**   | `SettingsScreen`, `ThemeSettingsScreen`, `LanguageSettingsScreen`, `NotificationSettingsScreen` |
| **BLoC**          | `SettingsBloc` — manages preferences with `data_local_storage`                                  |
| **Dependencies**  | `core_ui`, `core_localization`, `data_local_storage`, `data_analytics`                          |
| **Testing Focus** | BLoC tests for preference persistence; widget tests for settings toggles                        |

---

### 4.4 `feature_profile` — User Profile Management

| Aspect            | Detail                                                                                |
| ----------------- | ------------------------------------------------------------------------------------- |
| **Purpose**       | View/edit profile, avatar upload, account actions                                     |
| **Key Screens**   | `ProfileScreen`, `EditProfileScreen`                                                  |
| **BLoC**          | `ProfileBloc` — handles profile CRUD, avatar upload with `data_file_handling`         |
| **Dependencies**  | `core_ui`, `core_localization`, `data_network`, `data_file_handling`, `domain_models` |
| **Testing Focus** | BLoC tests for profile update flows; widget tests for form validation                 |

---

### 4.5 `feature_notifications` — Push & In-App Notifications

| Aspect             | Detail                                                                                                                                               |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Push notification handling, in-app notification center, notification preferences                                                                     |
| **Key Exports**    | `NotificationService`, `NotificationHandler`, `NotificationsScreen`                                                                                  |
| **BLoC**           | `NotificationsBloc` — manages notification list, mark-as-read, deep link routing                                                                     |
| **Design Pattern** | **Observer Pattern** — notification service broadcasts to multiple handlers; **Command Pattern** — notification actions are encapsulated as commands |
| **Dependencies**   | `core_ui`, `core_navigation`, `data_local_storage`                                                                                                   |
| **Testing Focus**  | Unit tests for notification parsing; BLoC tests for notification state; integration tests for deep linking                                           |

---

### 4.6 `feature_chat` — Real-Time Messaging

| Aspect             | Detail                                                                                                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | 1:1 and group messaging, real-time updates, message history, media sharing                                                                                                   |
| **Key Screens**    | `ChatListScreen`, `ChatScreen`, `ChatDetailScreen`                                                                                                                           |
| **BLoC**           | `ChatBloc` — handles message send/receive, pagination, real-time subscriptions                                                                                               |
| **Design Pattern** | **Repository Pattern** — abstract `ChatRepository` enables switching between backends (Supabase Realtime, Firebase, WebSocket); **Builder Pattern** for message construction |
| **Dependencies**   | `core_ui`, `core_localization`, `data_network`, `data_file_handling`, `domain_models`                                                                                        |
| **Testing Focus**  | BLoC tests with mock streams; widget tests for message bubbles; integration tests for real-time sync                                                                         |

---

### 4.7 `feature_media_gallery` — Image/Video Gallery & Viewer

| Aspect            | Detail                                                                 |
| ----------------- | ---------------------------------------------------------------------- |
| **Purpose**       | Grid/list galleries, full-screen viewer with zoom/swipe, media caching |
| **Key Screens**   | `GalleryScreen`, `MediaViewerScreen`                                   |
| **BLoC**          | `GalleryBloc` — handles media loading, pagination, caching             |
| **Dependencies**  | `core_ui`, `data_network`, `data_file_handling`                        |
| **Testing Focus** | Widget tests for gallery layouts; unit tests for caching logic         |

---

### 4.8 `feature_search` — Universal Search

| Aspect             | Detail                                                                                                           |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Search UI with suggestions, recent searches, filter chips, debounced input                                       |
| **Key Screens**    | `SearchScreen`                                                                                                   |
| **BLoC**           | `SearchBloc` — handles debounced search, suggestions, history                                                    |
| **Design Pattern** | **Strategy Pattern** — define `SearchDelegate<T>` interface, each feature provides its own search implementation |
| **Dependencies**   | `core_ui`, `core_utils` (Debouncer), `data_local_storage` (search history)                                       |
| **Testing Focus**  | BLoC tests for debounce and state transitions; widget tests for search UI                                        |

---

### 4.9 `feature_payments` — In-App Payments & Subscriptions

| Aspect             | Detail                                                                                              |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| **Purpose**        | Payment method management, checkout flow, subscription management, receipt handling                 |
| **Key Screens**    | `PaymentMethodsScreen`, `CheckoutScreen`, `SubscriptionScreen`                                      |
| **BLoC**           | `PaymentBloc` — handles payment processing, subscription status                                     |
| **Design Pattern** | **Strategy Pattern** — `PaymentGateway` interface with implementations for Stripe, RevenueCat, etc. |
| **Dependencies**   | `core_ui`, `core_localization`, `data_network`, `data_local_storage`                                |
| **Testing Focus**  | BLoC tests for payment flows; unit tests for price calculation; mock payment gateway tests          |

---

### 4.10 `feature_map` — Map & Location Services

| Aspect             | Detail                                                                                               |
| ------------------ | ---------------------------------------------------------------------------------------------------- |
| **Purpose**        | Map display, marker management, location search, directions, geofencing                              |
| **Key Screens**    | `MapScreen`, `LocationPickerScreen`                                                                  |
| **BLoC**           | `MapBloc` — handles map state, markers, user location                                                |
| **Design Pattern** | **Adapter Pattern** — `MapProvider` interface with Google Maps / Apple Maps / Mapbox implementations |
| **Dependencies**   | `core_ui`, `domain_models` (GeoLocation)                                                             |
| **Testing Focus**  | Unit tests for coordinate calculations; BLoC tests for marker management                             |

---

## 5. Cross-Cutting Packages

Packages that provide cross-cutting functionality used across all layers.

---

### 5.1 `pkg_dependency_injection` — Service Locator & DI

| Aspect             | Detail                                                                                                                             |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**        | Centralized dependency registration and resolution                                                                                 |
| **Key Exports**    | `ServiceLocator`, `Module` base class, module registration APIs                                                                    |
| **Design Pattern** | **Service Locator Pattern** — each package defines a `Module` that registers its dependencies; the app composes modules at startup |
| **Dependencies**   | `get_it`, `injectable` (optional)                                                                                                  |
| **Testing Focus**  | Unit tests for module registration; verify no missing dependencies                                                                 |

```dart
// Each package defines its own Module
class AuthModule extends Module {
  @override
  void register(ServiceLocator sl) {
    sl.registerLazySingleton<AuthRepository>(() => SupabaseAuthRepository(sl()));
    sl.registerFactory(() => AuthBloc(sl()));
  }
}

// App startup composes all modules
void configureDependencies() {
  CoreModule().register(sl);
  DataModule().register(sl);
  AuthModule().register(sl);
  // ...
}
```

---

### 5.2 `pkg_testing_utils` — Test Helpers & Fakes

| Aspect            | Detail                                                                                                                                                           |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**       | Shared test infrastructure used by all packages                                                                                                                  |
| **Key Exports**   | `Fakes` (FakeUser, FakeAuthToken, etc.), `MockApiClient`, `MockStorageService`, `TestHelpers` (pumpApp, pumpWidget with theme & localization), `BlocTestHelpers` |
| **Dependencies**  | `mocktail`, `bloc_test`, `flutter_test`                                                                                                                          |
| **Testing Focus** | N/A (this IS the testing package)                                                                                                                                |

---

## Summary Matrix

| #    | Package                    | Layer         | Design Pattern(s)                     | Priority    |
| ---- | -------------------------- | ------------- | ------------------------------------- | ----------- |
| 1.1  | `core_ui`                  | Core          | Token-based Design System             | 🔴 Critical |
| 1.2  | `core_utils`               | Core          | Result Pattern                        | 🔴 Critical |
| 1.3  | `core_localization`        | Core          | Delegate Pattern                      | 🟡 High     |
| 1.4  | `core_navigation`          | Core          | Strategy Pattern (Guards)             | 🔴 Critical |
| 2.1  | `data_network`             | Data          | Interceptor / Chain of Responsibility | 🔴 Critical |
| 2.2  | `data_local_storage`       | Data          | Repository + Adapter Pattern          | 🔴 Critical |
| 2.3  | `data_analytics`           | Data          | Adapter + Composite Pattern           | 🟡 High     |
| 2.4  | `data_file_handling`       | Data          | Command Pattern                       | 🟡 High     |
| 2.5  | `data_connectivity`        | Data          | Observer Pattern                      | 🟢 Medium   |
| 3.1  | `domain_models`            | Domain        | Value Object Pattern                  | 🔴 Critical |
| 3.2  | `domain_auth`              | Domain        | Repository Pattern                    | 🔴 Critical |
| 4.1  | `feature_auth`             | Feature       | BLoC Pattern                          | 🔴 Critical |
| 4.2  | `feature_onboarding`       | Feature       | State Machine Pattern                 | 🟡 High     |
| 4.3  | `feature_settings`         | Feature       | BLoC Pattern                          | 🟡 High     |
| 4.4  | `feature_profile`          | Feature       | BLoC Pattern                          | 🟡 High     |
| 4.5  | `feature_notifications`    | Feature       | Observer + Command Pattern            | 🟡 High     |
| 4.6  | `feature_chat`             | Feature       | Repository + Builder Pattern          | 🟢 Medium   |
| 4.7  | `feature_media_gallery`    | Feature       | BLoC Pattern                          | 🟢 Medium   |
| 4.8  | `feature_search`           | Feature       | Strategy Pattern                      | 🟢 Medium   |
| 4.9  | `feature_payments`         | Feature       | Strategy Pattern                      | 🟢 Medium   |
| 4.10 | `feature_map`              | Feature       | Adapter Pattern                       | 🟢 Medium   |
| 5.1  | `pkg_dependency_injection` | Cross-cutting | Service Locator Pattern               | 🔴 Critical |
| 5.2  | `pkg_testing_utils`        | Cross-cutting | —                                     | 🔴 Critical |

---

## Recommended Build Order

Build in **dependency order** — each phase unlocks the next.

```mermaid
graph LR
    P1[Phase 1: Core] --> P2[Phase 2: Domain]
    P2 --> P3[Phase 3: Data]
    P3 --> P4[Phase 4: Cross-cutting]
    P4 --> P5[Phase 5: Features]
```

| Phase       | Packages                                                                                | Rationale                                               |
| ----------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Phase 1** | `core_utils`, `core_ui`, `core_localization`                                            | Foundation — everything depends on these                |
| **Phase 2** | `domain_models`, `domain_auth`, `core_navigation`                                       | Business models and routing needed before data/features |
| **Phase 3** | `data_network`, `data_local_storage`, `data_connectivity`                               | Infrastructure for all features                         |
| **Phase 4** | `pkg_dependency_injection`, `pkg_testing_utils`, `data_analytics`, `data_file_handling` | Wire everything together; testing infrastructure        |
| **Phase 5** | All `feature_*` packages                                                                | Build features on top of solid foundation               |

---

## Testing Strategy

> [!TIP]
> Target **minimum 80% code coverage** on Core, Domain, and Data packages. Feature packages should target **70%+** with focus on BLoC tests.

| Layer         | Test Type                 | Tool                                    | Focus                                  |
| ------------- | ------------------------- | --------------------------------------- | -------------------------------------- |
| Core          | Unit Tests                | `flutter_test`                          | Pure functions, extensions, validators |
| Core UI       | Widget Tests + Goldens    | `flutter_test`, `golden_toolkit`        | Visual regression                      |
| Domain        | Unit Tests                | `test` (pure Dart)                      | Models, serialization, business rules  |
| Data          | Unit + Integration Tests  | `flutter_test`, `mocktail`              | Mock HTTP, verify interceptors         |
| Feature       | BLoC Tests + Widget Tests | `bloc_test`, `mocktail`, `flutter_test` | State transitions, UI interactions     |
| Cross-cutting | N/A (helpers only)        | —                                       | Used BY tests, not tested themselves   |

### Testing Patterns

- **BLoC Tests**: Use `blocTest()` from `bloc_test` — define `seed`, `act`, `expect` states
- **Repository Tests**: Mock data sources with `mocktail`; verify correct mapping
- **Widget Tests**: Use `pumpApp()` helper from `pkg_testing_utils` to inject theme + localization
- **Integration Tests**: Use `integration_test` package for end-to-end feature flows

---

## Git Submodule Usage

Each package lives in its own git repository. In your app, add them as submodules:

```bash
# Add a package as submodule
git submodule add git@github.com:your-org/core_ui.git packages/core_ui

# Reference in app's pubspec.yaml
dependencies:
  core_ui:
    path: packages/core_ui
```

### Folder Structure in App

```
my_app/
├── packages/               # git submodules live here
│   ├── core_ui/
│   ├── core_utils/
│   ├── core_navigation/
│   ├── data_network/
│   ├── feature_auth/
│   └── ...
├── lib/                    # app-specific code (composition root)
├── test/
└── pubspec.yaml
```

---

## Key Principles

1. **Single Responsibility** — each package does one thing well
2. **Interface Segregation** — expose abstract interfaces; implementations are swappable
3. **Dependency Inversion** — features depend on abstractions, not concrete implementations
4. **Testability First** — every package is independently testable with no Firebase/Supabase dependency in tests
5. **Minimal Public API** — use barrel files to control what's exported; hide implementation details
