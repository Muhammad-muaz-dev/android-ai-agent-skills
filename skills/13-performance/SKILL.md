---
name: performance
description: Improve Android app performance across startup, rendering, memory, battery, networking, database, threading, profiling, and regression prevention.
---

# Performance Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Performance Team / performance-skill agent

Purpose
-------
This Performance Skill defines a standardized, measurable, and maintainable approach for analyzing, optimizing, and protecting application performance across Android devices and OS versions. Its goal is to ensure apps are fast, responsive, memory- and battery-efficient, scalable, and regression-free while preserving architecture and correctness.

Scope
-----
Includes:
- Measurement & instrumentation (Perfetto, Firebase Performance, custom metrics)
- Reproducible benchmarking (startup, scrolling, CPU hotspots)
- Rendering & jank prevention
- Memory usage, leak detection, and mitigation
- Coroutine & concurrency best practices
- Network efficiency and caching patterns
- Database query optimization (Room/SQLite)
- Image loading and caching
- Background work and battery friendly scheduling
- App size reduction & resource optimization
- CI integration, regression detection, and monitoring

Excludes:
- Feature/API design decisions unrelated to performance
- Functional behavior changes except when required to fix performance defects

High-level Principles
---------------------
1. Measure first — identify real bottlenecks with reliable data before changing code.
2. Smallest safe change — prefer minimal, well-tested fixes with clear metrics.
3. Reproducibility — every optimization must include a reproducible benchmark or test.
4. Preserve architecture & readability — avoid hacks that reduce maintainability.
5. Platform-aware — consider Android version differences and device variability.
6. Test-driven performance — add tests and CI gates for critical performance paths.
7. Observe & guard — add monitoring and alerts to detect regressions early.

Key Responsibilities
--------------------
- Define performance targets (startup time, P95 frame latency, memory footprint, battery impact).
- Create measurement suite and capture baselines for critical flows.
- Identify hotspots using profiler traces and heap dumps.
- Implement targeted optimizations (lazy init, caching, efficient layouts).
- Add regression tests, instrumentation, and CI jobs.
- Provide rollout/rollback plan and monitor post-release.
- Document tradeoffs, expected improvements, and validation artifacts.

Measurement & Instrumentation
-----------------------------
- Always capture a baseline before changes.
- Tools:
  - Perfetto & Android Studio System Profiler for traces.
  - Firebase Performance or APM for network/latency metrics.
  - LeakCanary for leak detection in debug builds.
  - Benchmark library (androidx.benchmark) for microbenchmarks (startup, serialization).
  - Systrace and adb shell dumpsys for low-level data.
- Metrics to track:
  - Cold / warm / hot startup times (ms)
  - First Contentful Paint / time-to-interactive
  - Frame timing: dropped frames, 50th/95th/99th percentile frame time
  - Memory: resident set size (RSS), heap size, allocation rate
  - CPU: % usage during target flows
  - Network: request count, payload sizes, RTT, tail-latency
  - Battery / wakeup counts (for background work)
- Baselines:
  - Capture device/OS matrix and list of representative devices.
  - Store baseline traces and test artifacts in a performance artifact store for comparison.

Workflow (measure → optimize → validate)
----------------------------------------
1. Define objective & target metric(s).
2. Reproduce & capture baseline (Perfetto, Benchmark, LeakCanary).
3. Hypothesize root cause and prioritize low-risk fixes.
4. Implement minimal change with feature-flag or gated rollout if high-risk.
5. Validate with same measurement tools and compare to baseline.
6. Add regression tests and CI checks.
7. Deploy staged rollout; monitor telemetry and roll back if necessary.
8. Document results & lessons learned.

Startup Performance
-------------------
- Goals: reduce cold/warm/hot startup times and time-to-interactive.
- Patterns:
  - Avoid heavy work in Application.onCreate; defer or lazy-init components.
  - Use ContentProvider only for critical early init; prefer dynamic initialization.
  - Move non-critical initializations to background threads or lazy singletons.
  - Use androidx.startup for controlled initialization ordering if needed.
  - Defer network calls until after first frame; use preload caches if needed.
  - Optimize resource inflation and avoid complex view hierarchies on first screen.
- Tools:
  - androidx.benchmark for startup benchmarking: measure cold/warm/hot separately.
  - Perfetto traces with boot tracing for cold start analysis.

UI Rendering & Jank
-------------------
- Goals: stable 60fps (or device refresh rate) with minimal dropped frames.
- Common causes: heavy work on main thread, expensive measure/layout passes, large bitmaps decoded on UI thread.
- Patterns:
  - Keep main thread work < 16ms per frame.
  - Batch UI work; avoid synchronous disk/network calls.
  - Use hardware layers for animations where appropriate.
  - Reduce view hierarchy depth and avoid nested weights.
  - Use constraint layouts judiciously and prefer simple layouts.
  - Use RecyclerView with DiffUtil and ListAdapter for lists.
  - Offload expensive computations to background threads and use coroutines/Dispatchers.Default for CPU work.
- Diagnostics:
  - Use frame timeline in Android Studio or Perfetto GPU counters to find long frames.
  - Capture traces around UI transitions and inspect main thread stacks.

Memory Optimization & Leak Prevention
-------------------------------------
- Goals: stable memory footprint, avoid OOM and GC storms, no persistent leaks.
- Patterns:
  - Prefer application-scoped singletons that don't hold Activity/Context references.
  - Use viewLifecycleOwner for Fragment-scoped coroutines and listeners.
  - Unregister listeners and callbacks in lifecycle callbacks.
  - Use WeakReference where appropriate and lifecycle-aware components.
  - Use in-memory caches with bounded size and eviction (LruCache).
  - Resize and downsample bitmaps (use Coil/Glide/ Fresco with memory-conscious config).
- Tools:
  - LeakCanary in debug builds for leak detection.
  - Heap dump analysis (.hprof) in Android Studio Memory Analyzer: dominator tree, GC roots.
  - Allocation tracking in Android Studio profiler to detect allocation spikes.

RecyclerView & Lists
--------------------
- Use ListAdapter + DiffUtil to avoid notifyDataSetChanged.
- Reuse ViewHolders; avoid creating objects in onBindViewHolder.
- Use payloads for partial updates.
- Avoid heavy layout computations in onBindViewHolder; precompute on background threads.
- Use nested RecyclerViews only when justified — prefer view types or composables for complex layouts.
- For very large datasets, use Paging 3 with RemoteMediator when combining DB + network.

Image Loading & Media
---------------------
- Use modern libraries (Coil recommended for Kotlin + Compose; Glide/ Picasso alternatives).
- Configure disk and memory caches; use OkHttp caching for network images.
- Resize/transform images server-side when possible; otherwise request scaled sizes in image loader.
- Use placeholders and low-res progressive loading for smooth UX.
- Avoid decoding full-res on main thread; use decoder with sampling.
- For large GIFs/videos, avoid loading in memory; prefer streaming or lightweight previews.

Database (Room & SQLite)
------------------------
- Use Room with proper indexing to optimize queries; analyze queries with EXPLAIN QUERY PLAN.
- Avoid SELECT * when only a subset of columns is required.
- Use transactions and batch writes to amortize overhead.
- Use Paging 3 for large lists with DB-backed PagingSource or RemoteMediator.
- Run and test migration scripts on representative datasets; include migration tests in CI.

Network & Serialization
-----------------------
- Reduce chatty APIs: batch requests when possible.
- Use HTTP caching (OkHttp) honoring Cache-Control and ETag.
- Use compression (gzip) when supported by server.
- Avoid synchronous network calls on main thread; use Coroutines/Flow.
- Optimize serialization: choose Moshi / Kotlinx.serialization and avoid expensive reflection on critical paths.
- Use pagination & lazy loading for large datasets and media lists.
- Instrument request timings and payload sizes; alert on regressions.

Coroutines & Concurrency
------------------------
- Principles:
  - Use structured concurrency (avoid GlobalScope).
  - Prefer explicit dispatchers: Dispatchers.IO for IO, Dispatchers.Default for CPU-bound work.
  - Cancel coroutines promptly when UI is destroyed (use lifecycleScope/viewModelScope).
  - Avoid excessive context switches and unnecessary launch hierarchies.
  - Use SupervisorJob where parent shouldn't cancel children unexpectedly.
- Patterns:
  - Provide DispatcherProvider abstraction to make dispatchers testable.
  - Use withContext for blocking calls rather than launching new coroutines.
  - Use channels and sharedFlow/stateFlow intentionally, with backpressure in mind.

Background Work & Battery
-------------------------
- Use WorkManager for deferrable, guaranteed background work; set correct constraints and backoff policies.
- Use JobScheduler or foreground service only for critical, user-visible tasks.
- Batch work and avoid wake locks where possible; prefer push notifications to trigger on-demand work.
- Respect Doze and App Standby: schedule using WorkManager with appropriate constraints.
- Minimize frequent background location updates; use significant-change or geofencing APIs when possible.

APK / App Bundle Size
---------------------
- Use Android App Bundle (AAB) with Play Store splitting.
- Enable R8/ProGuard with conservative keep rules and resource shrinking.
- Remove unused resources and flavors; use vector drawables where appropriate.
- Modularize features into Dynamic Feature Modules to defer delivery.
- Avoid bundling large media files in APK; use remote content or on-demand downloads.

Testing, CI & Regression Detection
----------------------------------
- Incorporate performance checks into CI:
  - Run microbenchmarks (androidx.benchmark) and fail on regressions beyond threshold.
  - Run smoke UI performance tests on emulators for critical flows (startup, main list scroll).
  - Automate memory leak detection (instrumented tests that run LeakCanary assertions in CI-like flows).
- Store performance baselines and compare diffs; keep artifact store of traces and metrics.
- Add alerting on production telemetry (P95 frame time, crash rate, memory growth) for early detection.

Observability & Post-release Monitoring
---------------------------------------
- Instrument business-critical flows with lightweight, privacy-safe metrics (latency, error rates).
- Use APMs (Firebase Performance, Datadog, Sentry) and custom metrics to monitor regressions.
- Tag telemetry with release/build metadata to correlate regressions to releases.
- Establish SLOs/SLA for performance metrics and create alerts for breaches.

Tradeoffs & Documentation
------------------------
- Document tradeoffs for any optimization: complexity, maintainability, memory vs CPU, latency vs battery.
- Keep a short "why" and "how" in PR descriptions for performance changes, including before/after metrics.

Examples & Code Sketches
------------------------
- Startup benchmark (androidx.benchmark):
```kotlin
@LargeTest
class StartupBenchmark {
  @get:Rule val benchmarkRule = MacrobenchmarkRule()
  @Test fun startup() = benchmarkRule.measureRepeated(
    packageName = "com.example.app",
    metrics = listOf(StartupTimingMetric()),
    iterations = 10
  ) {
    pressHome()
    startActivityAndWait()
  }
}
