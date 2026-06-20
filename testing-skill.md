# Testing Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Testing Team / testing-skill agent

Purpose
-------
The Testing Skill defines a standardized, scalable, and maintainable approach to testing Android applications. It prescribes architecture, tools, conventions, and deliverables so tests are deterministic, fast in CI, and provide high confidence that features behave correctly across Android versions and environments.

Scope
-----
Includes:
- Test strategy, test pyramid, and test types (unit, integration, UI, E2E, performance, security, accessibility)
- Test architecture & DI patterns, test doubles, and fakes
- Tooling: JUnit, Robolectric, Espresso, Compose testing, MockWebServer, Mockito/MockK, androidx.benchmark, LeakCanary, Firebase Test Lab
- CI integration and gating: fast-unit pipeline + nightly/integration pipelines
- Test data strategy, fixtures, and replayable fixtures
- Flaky test prevention & detection, test artifact collection

Excludes:
- Changing production architecture solely to make tests simpler (exceptions require cross-team sign-off)

High-level Principles
---------------------
1. Test behavior, not implementation.
2. Follow the test pyramid: many unit tests, fewer integration tests, very few end-to-end tests.
3. Determinism: tests must be deterministic and isolated.
4. Fast unit tests in pull-request CI (<5s-30s); heavier integration/UI tests run in separate pipelines or nightlies.
5. Reproducible environments: use emulators, Robolectric, or containerized runners with fixed seeds.
6. Make tests part of the definition of done (green CI required).
7. Instrument test code and collect artifacts (logs, traces, screenshots) for diagnosis.

Testing Layers & Responsibilities
---------------------------------
- Unit Tests
  - Scope: pure logic, ViewModels, UseCases, utility classes.
  - Tools: JUnit5/JUnit4, MockK/Mockito, Turbine for Flow, kotlin.test.
  - Should be very fast, independent of Android SDK (use Robolectric only if Android API required).
- Integration Tests
  - Scope: repository + network + DB integration, mappers, paging.
  - Tools: MockWebServer, in-memory Room, Hilt with test components, Robolectric when necessary.
  - Use real JSON fixtures and validate boundary behavior.
- UI Tests (Instrumented)
  - Scope: activity/fragment flows, Compose UI, navigation, accessibility.
  - Tools: Espresso, Compose Test, UI Automator for system-level behavior.
  - Run on emulators/devices in CI or Firebase Test Lab.
- End-to-End (E2E)
  - Scope: high-impact user journeys on real devices/emulators (login, on-boarding, purchase).
  - Use device farms or staged pipelines; minimize number of E2E tests.
- Performance Tests
  - Scope: startup, scrolling, serialization; benchmarks using androidx.benchmark and Perfetto traces.
- Security & Accessibility Tests
  - Automated checks for common security misconfigurations and accessibility assertions (a11y).
- Regression & Canary Tests
  - Nightly or pre-release runs against representative device matrix and production-like data.

Test Design Conventions
-----------------------
- Arrange / Act / Assert pattern.
- Tests must be small and focused: one assert per assertion group.
- Avoid sleeps/time-based waits; use IdlingResources, Espresso IdlingRegistry, or explicit synchronization constructs.
- Use deterministic seeds for randomness.
- Prefer StateFlow/SharedFlow test utilities (Turbine) for reactive assertions.

Dependency Injection & Testability Patterns
------------------------------------------
- Use DI (Hilt recommended) with test components to inject fakes.
- Expose TestInstaller modules that bind fakes for networking, time providers, geolocation, and storage.
- Provide DispatcherProvider abstraction to make coroutine dispatchers testable.

Example: DispatcherProvider for tests
```kotlin
interface DispatcherProvider {
  val Main: CoroutineDispatcher
  val IO: CoroutineDispatcher
  val Default: CoroutineDispatcher
}
object TestDispatchers : DispatcherProvider {
  override val Main = UnconfinedTestDispatcher()
  override val IO = StandardTestDispatcher()
  override val Default = StandardTestDispatcher()
}