---
name: networking-api
description: Design and implement Android API/networking layers with Retrofit, OkHttp, serialization, DTO mapping, error handling, auth headers, caching, and tests.
---

# API Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: API Team / api-skill agent

## Purpose
This API Skill specifies a production-ready, scalable, secure, and maintainable networking layer for Android apps. It defines architecture, conventions, code patterns, and deliverables so any agent or developer can implement consistent, testable, and production-quality API integrations while keeping networking isolated from UI and business logic.

This document is the single source of truth for networking architecture, integration rules, error handling, caching, auth, performance, testing, and versioning.

## Scope
Covers:
- Retrofit + OkHttp configuration
- Authentication strategies and token refresh
- Serialization, DTO → domain mapping
- Error handling and typed network results
- Caching strategies & offline-first approaches
- Pagination (including Paging 3)
- Retry, backoff and throttling policies
- Security best practices (HTTPS, certificate pinning)
- Observability, logging guidelines
- DI integration (Hilt/Moshi/Kotlinx.serialization)
- Tests, lint rules and CI checks

Excludes:
- UI code, navigation, and business rules beyond routing decisions
- Database schema design (networking may provide caching strategy, not DB schema)
- Feature-specific endpoint semantics (these live in feature modules)

## High-level Principles
1. Single responsibility: networking code handles only API communication and transforms results into domain-friendly forms.
2. Explicit contracts: expose domain models to upper layers; keep DTOs internal.
3. Testability-first: make networking components (interceptors, authenticators, mappers) injectable and unit-testable.
4. Fail-safe: convert all errors into structured, non-throwing results.
5. Security by default: HTTPS, minimal sensitive logging, secure token storage.
6. Configurable environments: support dev/staging/production via compile-time/build-time configuration.
7. Backward compatibility: non-breaking graph changes allowed; breaking changes require migration steps.

## Architecture Overview
- Data Layer responsibilities:
  - Remote data source: Retrofit service interfaces + mappers
  - Local data source (optional): caching/DB abstraction (Room/Proto) — separate agent
  - Repository: orchestrates remote + local and exposes Flows/Result-wrapped domain models
- Layering:
  - API module (networking common): provides Retrofit module, OkHttp client factories, interceptors, Result wrappers, exceptions mapping utilities
  - Feature modules: provide Retrofit service interfaces and mappers; register graphs with DI
- DI: Hilt recommended — provide singletons for OkHttpClient, Retrofit, Json parser, AuthManager, and feature service bindings
- Concurrency: Kotlin Coroutines + Flow for streaming, suspend functions for one-shot calls

## Conventions & Naming
- Modules:
  - :network (common networking infra)
  - :feature_<name> (contains Retrofit interface and mappers)
- Retrofit service interface names: <Feature>NameApi e.g., ProfileApi
- DTO files: network/dto/<endpoint>_dto.kt
- Mappers: network/mappers/<dto>_mapper.kt (extensions: toDomain(), toDto())
- Repositories: domain-facing repository interfaces in domain module; networking implements the data layer
- Exceptions/Results: network/ApiResult.kt with sealed classes (Success/NetworkError/ApiError/UnknownError)

## Gradle / Build
- Put common networking dependencies into :network module.
- Use build-time config for base URLs via Gradle BuildConfig or a safe config manager.
- Safe Args not relevant here, but never embed secrets in BuildConfig; use secure runtime injection for secrets during CI/CD.

Example dependencies:
- Retrofit
- OkHttp + logging
- Moshi or Kotlinx.serialization (pick one; be consistent)
- Hilt
- Kotlin Coroutines
- Okio (for uploads/downloads)
- Paging 3 (if used)
- Retrofit-Kotlinx-serialization or Moshi converter

## Retrofit & OkHttp Patterns

Core OkHttpClient construction (conceptual):
- Connection pooling, timeouts, retryOnConnectionFailure false/true based on policy.
- Interceptors:
  - Application Interceptors:
    - NetworkAvailabilityInterceptor (optional) — short-circuits when offline
    - HeaderInterceptor — injects Accept, Content-Type, platform headers
    - AuthInterceptor — adds Authorization header (read-only)
  - Network Interceptors:
    - CachingInterceptor — respect caching headers or override for offline
    - LoggingInterceptor — enabled only in debug
- Authenticator (OkHttp Authenticator) for token refresh:
  - Handles 401 responses, invokes TokenRefresher, retries original request once.
  - Must be idempotent & thread-safe; avoid infinite refresh loops.
- Certificate pinning: optional config via CertificatePinner.

Retrofit:
- Build with converter (Moshi/Kotlinx), call adapter for coroutines (suspend).
- Use dedicated Retrofit instance per service group if baseUrls differ.
- Retrofit service methods should be suspend functions returning DTO or Response<DTO> when you need status inspection.

DI pattern:
- Provide factory methods: provideOkHttpClient(authenticator?, interceptors...), provideRetrofit(okHttp, baseUrl, converter)
- Feature modules bind their service interfaces via @Provides or @Binds.

Example design: keep an ApiClientFactory in :network that can create Retrofit instances for dynamic base URLs.

## Authentication & Token Refresh

Principles:
- Auth logic (storage, refresh) belongs to API Skill but must only produce tokens; business decisions (logout) are signalled to higher layers via event.
- Secure token storage: EncryptedSharedPreferences/Android Keystore-backed storage.
- Token lifecycle:
  - AuthInterceptor adds bearer token if available.
  - Authenticator performs refresh on 401 with proper synchronization (single refresh at a time).
  - If refresh fails (invalid creds), propagate AuthRequired error to caller; higher layers handle sign-out/reauth.

Refresh implementation constraints:
- Use refresh token endpoint with exponential backoff / rate limiting.
- Limit retry attempts; on repeated failure, raise AuthFailure event (do not loop).
- Do not persist raw tokens in logs.

## DTO → Domain mapping
- All network DTOs must be mapped to domain models before returning to repository consumers.
- Mapping rules:
  - Keep DTOs as data classes in network module.
  - Provide extension functions toDomain() inside feature modules.
  - Validate required fields and either map to error results or provide safe defaults as per API Skill rules (avoid surprising nulls).

## Result & Error Handling
- Use a typed result wrapper — ApiResult / Either / Resource — standardized across app.
- Recommended sealed class:

```kotlin
sealed class ApiResult<out T> {
  data class Success<T>(val value: T): ApiResult<T>()
  data class ApiError(val code: Int, val body: ApiErrorBody?): ApiResult<Nothing>()
  data class NetworkError(val cause: IOException): ApiResult<Nothing>()
  data class AuthError(val message: String?): ApiResult<Nothing>()
  data class UnknownError(val throwable: Throwable): ApiResult<Nothing>()
}
```

- All public networking APIs should return either:
  - suspend fun foo(...): ApiResult<Domain>
  - fun fooFlow(...): Flow<ApiResult<Domain>>
- Mapping:
  - HTTP 2xx → Success (map DTO to domain)
  - HTTP 401 → AuthError (or attempt refresh via Authenticator and then re-evaluate)
  - HTTP 4xx → ApiError (map body to meaningful error model)
  - HTTP 5xx → ApiError with server code
  - IO / Timeout → NetworkError
  - Serialization exceptions → UnknownError / ApiError with parse details

- Avoid throwing exceptions across repository boundaries.

## Caching & Offline Strategies
- Respect server cache headers (Cache-Control, ETag).
- Use OkHttp cache for small caches; for larger or structured caching use local DB (Room) with an offline-first repository pattern:
  - Repository exposes: Flow<Resource<Domain>> that emits from DB then attempts network refresh and updates DB.
- Use Cache-Control short-circuits on poor network.
- For file downloads/uploads: use streams, resume if server supports Range.

## Pagination
- Support offset, page, and cursor styles.
- Prefer integrating Paging 3 for list flows:
  - Provide remote mediator or PagingSource implementation that uses Retrofit DTOs and maps to domain models.
- Provide standardized page parameters and response wrappers across features.

## Retry & Backoff
- Implement retry policies in a dedicated RetryInterceptor or use resiliency library:
  - Exponential backoff with jitter
  - Max retries (configurable)
  - Only retry safe idempotent requests by default (GET/PUT); avoid retrying non-idempotent POSTs unless server supports idempotency keys.
- For refresh flows, ensure single-refresh concurrency semantics.

## Multipart Uploads & Downloads
- Implement streaming uploads with RequestBody and progress callbacks if needed.
- Downloads: stream to disk via Okio/OkHttp and write to file with proper lifecycle/cancellation handling.

## Logging & Observability
- Development: use HttpLoggingInterceptor at BODY level for dev builds only.
- Production: minimal logging — log timings, status codes, endpoints (no PII or tokens).
- Metrics: instrument average latency, failure rates, and error distribution using telemetry (e.g., Sentry, Datadog). Keep tokens out of telemetry.
- Structured logs: include request id / correlation IDs when available.

## Security
- Enforce HTTPS on all endpoints.
- Avoid embedding secrets in repo or BuildConfig.
- Use EncryptedSharedPreferences / Android Keystore for tokens.
- Consider certificate pinning for high-security apps and provide clear upgrade/migration plan.
- Validate server-provided redirects/URIs before following.
- Sanitize / redact any PII before logging.

## Dynamic Feature Support & Missing Modules
- If a feature graph calls an endpoint in a dynamic module not installed, the API layer should fail gracefully with MissingFeature error and allow upper layers to trigger install flow.

## Testing Strategy
- Unit tests:
  - Test mappers, interceptors, authenticators using MockWebServer.
  - Test error mapping using Response scenarios and malformed payloads.
- Integration tests:
  - End-to-end tests with MockWebServer for full repository layer.
- Instrumentation:
  - Test uploads/downloads, paging integration.
- CI:
  - Run static analysis and network tests in CI (unit + integration).
  - Fail build on missing mapping or exposing DTOs to UI (lint rules).
- Provide test fixtures and sample MockWebServer responses per feature.

## Linting & CI Checks
- Lint rules:
  - Disallow Retrofit models leaking into ViewModel/UI packages.
  - Detect injection of OkHttp/Retrofit into ViewModels (prefer repositories).
  - Ensure Safe logging policy compliance (no tokens in logs).
- CI:
  - Run unit tests, run API contract tests (if contract available), run static checks.

## Versioning & Migrations
- API Skill spec follows semver.
- For API surface changes (field renames, arg types) provide migration guides.
- Maintain CHANGELOG.md in :network module.

## Deliverables (per integration)
- Retrofit service interfaces (suspend functions) with javadoc.
- OkHttp configuration and interceptors.
- Authentication strategy & TokenRefresher implementation.
- DTOs and mappers (toDomain()/toDto()).
- ApiResult sealed classes and extension mapping utilities.
- Paging3 adapters or PagingSource/RemoteMediator.
- Tests: unit + integration + sample MockWebServer fixtures.
- Documentation: endpoints table, expected args, error mapping.
- Checklist and migration notes.

## Checklist (pre-merge)
- [ ] Retrofit + OkHttp provided via DI
- [ ] Authentication in place with secure storage
- [ ] Token refresh handled with Authenticator & concurrency guard
- [ ] All responses mapped to ApiResult
- [ ] DTO → Domain mappers present & used
- [ ] Paging 3 integration for lists (if applicable)
- [ ] Unit & integration tests exist
- [ ] Logging disabled / sanitized for production
- [ ] Certificate pinning and HTTPS enforced if required
- [ ] No network code in ViewModels or UI layers
- [ ] CI runs networking tests & lint rules

## Example Patterns (concise)

OkHttp + Retrofit provider (conceptual example):

```kotlin
// network/NetworkModule.kt (Hilt)
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
  @Provides @Singleton
  fun provideOkHttpClient(
    authInterceptor: AuthInterceptor,
    logging: HttpLoggingInterceptor,
    authenticator: TokenAuthenticator,
    cache: Cache?
  ): OkHttpClient = OkHttpClient.Builder()
    .connectTimeout(15, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .addInterceptor(authInterceptor)
    .addNetworkInterceptor(CachingInterceptor())
    .authenticator(authenticator)
    .apply { if (cache != null) cache(cache) }
    .apply { if (BuildConfig.DEBUG) addInterceptor(logging) }
    .build()

  @Provides @Singleton
  fun provideRetrofit(okHttp: OkHttpClient, @BaseUrl baseUrl: String, moshi: Moshi): Retrofit =
    Retrofit.Builder()
      .baseUrl(baseUrl)
      .client(okHttp)
      .addConverterFactory(MoshiConverterFactory.create(moshi))
      .build()
}
```

ApiResult mapping helper:

```kotlin
suspend fun <T : Any, R> safeApiCall(call: suspend () -> Response<T>, mapper: (T) -> R): ApiResult<R> {
  return try {
    val response = call()
    if (response.isSuccessful) {
      val body = response.body()
      if (body != null) ApiResult.Success(mapper(body)) else ApiResult.UnknownError(IllegalStateException("Empty body"))
    } else {
      val errorBody = response.errorBody()?.string()
      ApiResult.ApiError(response.code(), parseApiError(errorBody))
    }
  } catch (io: IOException) {
    ApiResult.NetworkError(io)
  } catch (t: Throwable) {
    ApiResult.UnknownError(t)
  }
}
```

Authenticator sketch:

```kotlin
class TokenAuthenticator @Inject constructor(
  private val tokenRepository: TokenRepository,
  private val apiService: AuthApi
) : Authenticator {
  private val mutex = Mutex()

  override fun authenticate(route: Route?, response: Response): Request? = runBlocking {
    // Prevent parallel refreshes
    mutex.withLock {
      val currentToken = tokenRepository.getAccessToken()
      if (response.request.header("Authorization") == "Bearer $currentToken") {
         val refreshResult = runCatching { apiService.refreshToken(tokenRepository.getRefreshToken()) }
         if (refreshResult.isSuccess) {
           val newToken = refreshResult.getOrNull()!!.accessToken
           tokenRepository.saveAccessToken(newToken)
           return@runBlocking response.request.newBuilder()
             .header("Authorization", "Bearer $newToken")
             .build()
         } else {
           tokenRepository.clear()
           return@runBlocking null // give up - will propagate auth error
         }
      } else {
        // Another thread refreshed already, retry with new token
        val updated = tokenRepository.getAccessToken() ?: return@runBlocking null
        return@runBlocking response.request.newBuilder().header("Authorization", "Bearer $updated").build()
      }
    }
  }
}
```

## Governance & Ownership
- API Skill owned by API Team (or designated owner).
- Changes require PR with tests, migration notes, and CI green.
- Maintain a changelog and api-skill/README.md in :network module.

---

Add this file to your repository (recommended path: docs/api-skill.md or modules/network/README.md). If you want, I will next:
- Generate a concrete :network module with Hilt, Retrofit, OkHttp interceptors, TokenAuthenticator, ApiResult, and unit tests (MockWebServer) for a specific feature.
- Produce a sample feature module that demonstrates DTOs, mappers, repository, and Paging3 integration.

Tell me which feature or endpoint you want scaffolded and I'll generate the code and tests next.
