---
name: firebase
description: Integrate Firebase services in Android apps including Auth, Firestore, Realtime Database, Storage, FCM, Remote Config, Crashlytics, Performance, and security rules.
---

# Firebase Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Firebase Team / firebase-skill agent

## Purpose
This Firebase Skill defines a production-ready, secure, scalable, and maintainable approach to integrating Firebase services into Android applications. It prescribes architecture, conventions, patterns, and required deliverables so any agent or developer can implement consistent Firebase integrations while keeping Firebase concerns isolated inside the data layer.

This file is the single source of truth for Firebase work: Authentication, Firestore, Realtime Database (when needed), Storage, Cloud Messaging, Remote Config, Crashlytics, Performance, App Check, Dynamic Links, security rules, offline sync, and observability.

## Scope
Includes:
- Firebase SDK selection & configuration
- Authentication flows and session management
- Cloud Firestore design patterns, queries & security rules
- Realtime Database for specialized realtime use-cases
- Firebase Storage uploads/downloads and security
- FCM token management and message handling
- Remote Config and feature flags
- Crashlytics and Performance monitoring
- App Check, Dynamic Links
- Offline-first strategies, conflict resolution
- Mappers from Firebase models to domain models
- Testing, lint, CI and governance

Excludes:
- Business logic, UI code, navigation logic, or non-Firebase network integrations
- Database schema design outside Firebase products (except caching patterns that require coordination with DB teams)

## High-level Principles
1. Single responsibility — Firebase layer handles only Firebase interactions and mapping to domain models.
2. Separation of concerns — Firebase SDK objects (DocumentSnapshot, Task, etc.) must NOT leak to upper layers; expose domain models or typed wrappers.
3. Testability — Provide abstractions and inject Firebase dependencies so they are mockable/fakeable for tests.
4. Security-first — Use App Check, least privilege security rules, encrypted storage for secrets, and never log PII or tokens.
5. Offline-first where appropriate — Favor local caching + synchronization for resilient UX.
6. Configurable & environment-aware — Support dev/stage/prod projects, restrict sensitive features in dev.
7. Backward-compatible changes — Avoid breaking client behavior; when unavoidable provide migration steps.

## Module & Naming Conventions
- Modules:
  - :firebase (shared Firebase infra utilities, DI providers, common wrappers)
  - :feature_<name> (feature-level logic using Firebase via repository)
- Packages:
  - com.company.firebase (infra)
  - com.company.feature.<name>.data.firebase (feature integration)
- Class naming:
  - Repositories: <Feature>NameFirebaseRepository
  - Mappers: FirestoreUserMapper, StorageUploadMapper
  - Wrappers: FirebaseResult<T>, FirebaseError
- Resources: store Firebase config (google-services.json) per build flavor/environment and never commit production secrets into VCS.

## Recommended SDK & Tooling
- Use latest stable Firebase BOM (Bill of Materials).
- Core libs: firebase-auth, firebase-firestore, firebase-database (optional), firebase-storage, firebase-messaging, firebase-crashlytics, firebase-remote-config, firebase-perf, firebase-appcheck, firebase-dynamic-links, firebase-analytics.
- Kotlin Coroutines and Flow for asynchronous APIs.
- Hilt (or other DI) for injection.
- Use Kotlinx.serialization or Moshi for local parsing where needed.
- Use Mockable wrappers or Fake Firebase implementations for unit tests.

## Architectural Patterns
- Layers:
  - Firebase Infra (:firebase): DI module, wrappers (FirebaseResult), mappers, security helpers
  - Feature Data: feature_x FirebaseRepository implements domain repository interface
  - Domain: pure business interfaces & models
  - UI: receives domain models, observes Flows or LiveData
- Avoid injecting Firebase SDK into ViewModels; inject repositories or domain-facing interfaces.
- Provide short, focused repository methods (suspend functions or Flows) that return FirebaseResult/ApiResult or Result<T> wrappers.

## FirebaseResult (Error handling wrapper)
Use a sealed type to standardize results:

```kotlin
sealed class FirebaseResult<out T> {
  data class Success<T>(val value: T) : FirebaseResult<T>()
  data class Failure(val error: FirebaseError) : FirebaseResult<Nothing>()
}

data class FirebaseError(
  val code: FirebaseErrorCode,
  val message: String?,
  val cause: Throwable? = null,
  val metadata: Map<String, Any?> = emptyMap()
)

enum class FirebaseErrorCode {
  NETWORK, AUTH, PERMISSION_DENIED, NOT_FOUND, CONFLICT, QUOTA_EXCEEDED, PARSE_ERROR, UNKNOWN
}
```

All public Firebase repository methods should return FirebaseResult<T> or Flow<FirebaseResult<T>>.

## Dependency Injection & Initialization
- Provide a single place to configure Firebase clients and helpers (module :firebase).
- Example Hilt provision responsibilities:
  - FirebaseAuth (or an AuthManager abstraction)
  - FirebaseFirestore (with settings)
  - FirebaseStorage
  - FirebaseMessaging / FCM token manager
  - RemoteConfig
  - AppCheck provider setup (Play Integrity, SafetyNet, or debug provider for dev)
- Initialize Firebase in Application class but keep feature-specific initialization lazy/injected.

## Authentication Patterns
Support:
- Email/password, Google Sign-In, Anonymous, Phone, Custom auth, OAuth providers, account linking.

Patterns:
- Provide an AuthManager abstraction wrapping FirebaseAuth. Methods should be suspend-friendly and return FirebaseResult.
- Keep token/session storage secure (EncryptedSharedPreferences / Android Keystore).
- Provide session expiration awareness (listen to FirebaseAuth.authState).
- Provide an observable auth state Flow<UserSession?> for other agents to observe (but don't implement business decisions).
- For sensitive operations, re-authenticate before executing when required.

AuthManager example responsibilities:
- signInWithEmail(email, password): FirebaseResult<User>
- signOut(): FirebaseResult<Unit>
- linkWithCredential(provider, credential): FirebaseResult<User>
- sendPasswordReset(email)
- observeAuthState(): Flow<UserSession?>

Do not put business logic (e.g., onboarding completion) inside AuthManager.

## Cloud Firestore: Design & Best Practices
Design principles:
- Collections → documents. Favor flatter structures for scale.
- Use small documents (avoid >1MB documents).
- Choose collection/document IDs: prefer server-generated IDs for lists, meaningful stable IDs for singletons.
- Avoid extremely deep nesting. Use subcollections when logically needed.
- Model relationships with references or denormalized data for read-heavy paths.
- Indexing: declare composite indexes for complex queries; keep index size considerations in mind.

Queries & pagination:
- Use limit + startAfter for pagination (cursor-based) rather than offset when possible.
- Use snapshots and serverTimestamp where appropriate.
- Use security rules-aware queries (rules may reject queries that would expose data).

Listeners:
- Convert listeners to Flow using callbackFlow or channelFlow.
- Close listeners in coroutines scope on cancellation.

Example Flow wrapper:

```kotlin
fun <T> Query.asFlow(mapper: (DocumentSnapshot) -> T): Flow<FirebaseResult<List<T>>> = callbackFlow {
  val listener = addSnapshotListener { snapshot, error ->
    if (error != null) trySend(FirebaseResult.Failure(...))
    else {
      val mapped = snapshot?.documents?.mapNotNull { d -> mapper(d) }
      trySend(FirebaseResult.Success(mapped))
    }
  }
  awaitClose { listener.remove() }
}
```

Transactions & batched writes:
- Use transactions for atomic read-modify-write where necessary.
- Use WriteBatch for multi-document writes to reduce round trips.
- Limit size and cost of transactions; keep them short.

Field-level security and validation:
- Use serverTimestamp and security rules to validate client writes.
- Validate client-provided fields in security rules to avoid malicious data.

Firestore indexes and cost:
- Document reads are billable; design reads carefully, use projection queries where supported to avoid reading large fields.
- Cache locally to reduce reads when appropriate.

## Firestore Security Rules (Governance)
- Principles:
  - Least privilege.
  - Validate inputs (types, lengths, allowed values).
  - Use custom claims where applicable to store roles.
  - Prefer server-side enforcement for sensitive rules (Cloud Functions).
- Maintain rules in repo with versioning and tests (emulator).
- Use the Firebase Emulator Suite for rule testing and CI contract tests.

## Realtime Database (When to use)
- Use Realtime Database for extremely low-latency, chat-like or presence scenarios.
- Design with sharding and shallow structures to reduce bandwidth and costs.
- Use onDisconnect handlers for presence handling.
- Apply strict security rules, test with emulator.

## Firebase Storage (Uploads & Downloads)
- Use resumable uploads when large files; leverage firebase-storage SDK streams or OkHttp if using signed URLs.
- Validate file types and sizes before upload (client-side, but enforce with security rules).
- Use Storage Security Rules to restrict access (based on auth.uid or custom claims).
- For downloads, stream to disk with coroutine cancellation support and progress reporting.
- Use content-hash or versioned paths to support cache invalidation.

## Firebase Cloud Messaging (FCM)
- Provide a FcmManager abstraction to handle:
  - Registration & token lifecycle (refresh and secure storage of token if needed)
  - Topic subscriptions and unsubscriptions
  - Foreground & background handling (deliver data messages to repository or domain)
  - Notification channel management (Android 8+)
  - Handling of delivered messages with appropriate routing (UI or background sync)
- Security: never send sensitive PII in notifications; prefer server to send minimal payloads and let app fetch details via secure endpoints.
- When message directs to a destination, validate deep link and auth state before routing.

## Remote Config & Feature Flags
- Provide RemoteConfigManager abstraction:
  - Fetch & activate values with sensible defaults.
  - Expose typed getters (boolean, int, string).
  - Support staged rollouts and percent-based flags using Remote Config variants or server-side flags.
  - Integrate with feature-flag service if needed.
- Cache values and respect fetch intervals; support forced fetch in critical situations.

## Crashlytics & Performance Monitoring
- Integrate Crashlytics to capture crashes & non-fatal exceptions; attach non-sensitive diagnostic context (feature version, variant).
- Use custom keys sparingly; never include PII.
- Use Performance Monitoring to instrument traces: app startup, screen render times, expensive DB sync tasks, network call durations (if desired).
- Avoid over-instrumentation to reduce overhead.

## App Check & Security
- App Check should be enabled for production to reduce abuse of Firebase services.
- Use Play Integrity or SafetyNet production providers; use debug providers for development.
- Provide a toggle or build-flavor-safe switch to avoid blocking dev workflows.

## Dynamic Links
- Use dynamic links for onboarding invites or deep links; validate their payload and handle deferred deep linking flow (store pending link during auth or onboarding).
- Use a safe resume/resolution mechanism after sign-in or install.

## Offline Support & Conflict Resolution
- Firestore offers offline persistence — enable per-app policy where appropriate.
- For local-first UX:
  - Use local cache + eventual sync pattern.
  - Ensure conflicts resolved deterministically (server timestamps, last-write-wins) or design CRDT/merge logic in Cloud Functions for complex merges.
- Test conflict scenarios in emulator and integration tests.

## Migrations, Data Evolution & Versioning
- Apply schema evolution via versioned document fields or migration scripts (Cloud Functions or server-side migrations).
- Avoid destructive client-side migrations; coordinate with backend and analytics teams.
- Keep changelog and migration plan in repo.

## Observability & Logging
- Log only metadata (event names, status codes, non-sensitive identifiers).
- Redact or avoid logging tokens, PII, or sensitive payloads.
- Send higher-level metrics to telemetry (crash rate, sync latency, listener counts).
- Use correlatable IDs (request/operation IDs) for debugging cross-system flows.

## Testing Strategy
- Unit tests:
  - Wrap Firebase interactions using interfaces and test with fakes or spies.
  - Use Firebase local emulator for Firestore, Auth, Storage, and Functions where possible.
- Integration tests:
  - Use Firebase Emulator Suite to run CI tests verifying security rules and flows.
- Instrumentation tests:
  - Test FCM handling, downloads, and Storage flows on emulators or devices.
- Contract tests:
  - Keep sample payload fixtures and test failures for invalid/malformed data.
- CI:
  - Run emulator-based tests in CI where possible; fail on security rule regressions.

## Linting & CI Checks
- Enforce:
  - No Firebase SDK types exposed to UI/domain modules.
  - No logging of raw snapshots or Task results.
  - All public repository methods return FirebaseResult or Flow-wrapped results.
- Configure CI to run emulator tests, unit tests, static analysis, and rule checks for security rules.

## Governance & Ownership
- Firebase Skill is owned by Firebase Team / designated owner.
- Changes to rules, security policies, or telemetry must be PR-reviewed and include:
  - Tests (emulator) and migration notes.
  - Risk analysis (cost, security, data privacy).
- Maintain CHANGELOG.md and a security/rule-deployment checklist.

## Deliverables (for each integration)
- Firebase architecture doc & diagram for the feature.
- Auth flow design and AuthManager implementation.
- Firestore / Realtime DB data model and indexes.
- Storage policies & upload/download helpers.
- FCM handling design & token management.
- Remote Config plan & defaults.
- Crashlytics context keys and error-reporting guidance.
- App Check configuration notes.
- Security Rules with tests (emulator) and versioning.
- Offline & sync strategy and conflict resolution.
- Mapping utilities (snapshot → domain) and examples.
- Unit & emulator integration tests.
- CI checks and lint rules.
- Migration plan for breaking changes.

## Pre-merge Checklist
- [ ] Firebase dependencies centralized via BOM and properly versioned.
- [ ] All Firebase interactions wrapped in repositories/abstractions.
- [ ] No Firebase SDK types leaked above data layer.
- [ ] Security Rules implemented and covered by emulator tests.
- [ ] Auth flows and re-authentication behavior documented and implemented.
- [ ] Remote Config default values present.
- [ ] Crashlytics & Performance minimal instrumentation added.
- [ ] App Check configured for production.
- [ ] Sensitive data not logged; telemetry policy enforced.
- [ ] Offline and sync behavior documented & tested.
- [ ] CI runs emulator-based tests and lints Firebase code patterns.

## Example Implementations (concise)

Hilt provider (infra):

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object FirebaseModule {
  @Provides
  @Singleton
  fun provideFirebaseAuth(): FirebaseAuth = Firebase.auth

  @Provides
  @Singleton
  fun provideFirestore(): FirebaseFirestore {
    val db = Firebase.firestore
    val settings = FirebaseFirestoreSettings.Builder()
      .setPersistenceEnabled(true) // configurable per feature
      .build()
    db.firestoreSettings = settings
    return db
  }

  @Provides
  @Singleton
  fun provideStorage(): FirebaseStorage = Firebase.storage
}
```

AuthManager sketch:

```kotlin
interface AuthManager {
  val authState: Flow<UserSession?>
  suspend fun signInWithEmail(email: String, password: String): FirebaseResult<UserSession>
  suspend fun signOut(): FirebaseResult<Unit>
  suspend fun linkWithGoogle(idToken: String): FirebaseResult<UserSession>
}

class FirebaseAuthManager @Inject constructor(
  private val auth: FirebaseAuth
): AuthManager {
  override val authState: Flow<UserSession?> = callbackFlow {
    val listener = FirebaseAuth.AuthStateListener { firebaseAuth ->
      trySend(firebaseAuth.currentUser?.toUserSession())
    }
    auth.addAuthStateListener(listener)
    awaitClose { auth.removeAuthStateListener(listener) }
  }

  override suspend fun signInWithEmail(email: String, password: String): FirebaseResult<UserSession> {
    return try {
      val result = auth.signInWithEmailAndPassword(email, password).await()
      val user = result.user?.toUserSession()
      if (user != null) FirebaseResult.Success(user) else FirebaseResult.Failure(...)
    } catch (e: Exception) {
      FirebaseResult.Failure(mapException(e))
    }
  }
}
```

Firestore snapshot -> domain mapper pattern:

```kotlin
fun DocumentSnapshot.toDomainUser(): Result<User> {
  return try {
    val dto = this.toObject(UserDto::class.java) ?: return Result.failure(...)
    Result.success(dto.toDomain())
  } catch (e: Exception) {
    Result.failure(e)
  }
}
```

Listener -> Flow example:

```kotlin
fun DocumentReference.asFlow(mapper: (DocumentSnapshot) -> T): Flow<FirebaseResult<T>> = callbackFlow {
  val listener = addSnapshotListener { snapshot, error ->
    if (error != null) trySend(FirebaseResult.Failure(mapException(error)))
    else if (snapshot != null) {
      try {
        val mapped = mapper(snapshot)
        trySend(FirebaseResult.Success(mapped))
      } catch (e: Exception) { trySend(FirebaseResult.Failure(mapException(e))) }
    }
  }
  awaitClose { listener.remove() }
}
```

Security rules testing example:
- Add emulator-based unit tests for rules using the Firebase Emulator SDK; fail CI on regressions.

## Migration & Versioning
- Use semantic versioning for this skill.
- Include migration guides for breaking changes (security rule updates, collection renames).
- Prefer additive, backward-compatible changes.
- Coordinate with backend teams on field removals or contract changes.

---

