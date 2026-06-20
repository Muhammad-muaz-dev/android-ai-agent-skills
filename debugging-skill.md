# Debugging Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Debugging Team / debugging-skill agent

Purpose
-------
This Debugging Skill defines the standardized, evidence-driven approach for diagnosing, analyzing, and resolving bugs, crashes, performance regressions, ANRs, memory leaks, and unexpected behavior in Android applications.

It exists to:
- Ensure root-cause analysis (RCA) is systematic, reproducible, and minimally invasive.
- Provide diagnosable, testable, and low-risk fixes.
- Integrate with CI, observability, and release processes for continuous quality improvement.

Scope
-----
Covers:
- Crash & exception analysis (stack traces, tombstones)
- ANR diagnosis
- Memory leak detection and heap analysis
- Performance profiling (CPU, UI jank, startup)
- Concurrency & threading issues
- Network failure analysis
- Database & migration issues (Room/SQLite)
- Dependency injection problems
- Navigation & permission failures as they relate to stability
- Observability & telemetry-driven debugging workflows
- Automated triage and prioritization guidance

Excludes:
- Feature redesigns not required to fix a bug
- Unrelated architectural rewrites without documented benefits

High-level Principles
---------------------
1. Evidence-first: collect logs, traces, and metrics before proposing changes.
2. Smallest safe change: prefer targeted fixes with regression tests.
3. Preserve architecture: keep modifications minimal and aligned with project patterns.
4. Reproducible diagnosis: reproduce locally or in CI/emulator when possible.
5. Automated checks: add tests and CI gates to prevent regressions.
6. Observability-driven: prefer solutions that increase visibility (structured logs, traces, metrics).

Responsibilities (Agent must)
-----------------------------
- Triage incoming incidents and classify severity and impact.
- Collect required artifacts (logs, stack traces, heap dumps, ANR traces, systrace, Perfetto).
- Reproduce or simulate the issue in a controlled environment.
- Identify the root cause with evidence and mapping to source code.
- Propose the minimum-risk fix and apply it in a PR with tests.
- Validate the fix across affected Android API levels and configurations.
- Add regression tests, monitoring, and post-deploy verification.
- Document RCA, fix rationale, and prevention steps.

Debugging Workflow (step-by-step)
---------------------------------
1. Intake & Triage
   - Collect: incident id, user reports, device/OS, app version, build variant, reproducer steps, screenshots.
   - Classify severity: P0/P1/P2 and assign an owner.
2. Artifact Collection
   - Gather logs (logcat), crash reports (Sentry/Crashlytics/Play Console), stack traces, ANR traces (traces.txt), tombstones, Perfetto/systrace, heap dumps, network logs, DB snapshots.
3. Reproduce
   - Reproduce locally on emulator/device or create deterministic test using MockWebServer/fixtures or emulator settings.
4. Root Cause Analysis
   - Analyze artifacts, map stack frames to source, inspect related commits and regressions, and narrow to minimal repro.
   - Validate hypotheses with targeted instrumentation or additional logs.
5. Fix Design
   - Choose smallest safe change, write fix, add unit/integration/UI tests.
6. Risk Assessment
   - List side effects, compatibility with Android versions, performance impacts, and security considerations.
7. Implementation & Review
   - Submit PR with description, tests, and monitoring changes.
8. Verification
   - Run automated tests, QA, and dogfood builds; monitor post-deploy.
9. Postmortem & Prevention
   - Publish RCA, add CI checks or lint rules, and update documentation.

Required Evidence & Artifacts
----------------------------
For each incident collect one or more of:
- Crashlytics / Sentry report (full stack + breadcrumbs)
- Full logcat covering crash window
- ANR traces.txt from device or Play Console
- Tombstone from adb pull /data/tombstones/
- Perfetto or systrace capture
- Heap dump (.hprof) and LeakCanary traces
- MockWebServer logs / network fixtures
- Database snapshots or sample records
- Device population stats (percent of devices affected)
- User repro steps or automated reproducer tests

Diagnosis & Analysis Patterns
-----------------------------
- Stack trace mapping:
  - Symbolicate native traces if needed.
  - Expand inlined frames via debug symbols; map obfuscated names using ProGuard/R8 mappings.
- ANR:
  - Inspect traces.txt, find blocked main thread, long operations, or deadlocks.
  - Correlate with logs around the timestamp.
- Memory:
  - Use heap dumps and dominator tree to find GC roots, retainers, and leak suspects.
  - Confirm leaks with LeakCanary or repeated scenario.
- Performance:
  - Use Perfetto/Android Studio Profiler to find jank, frame drops, and CPU hotspots.
  - Use startup trace to split cold/warm start times.
- Concurrency:
  - Look for shared mutable state, missing synchronization, mis-scoped coroutines, or improper dispatcher usage.
- Network:
  - Reproduce with MockWebServer; inspect HTTP responses, error codes, timeouts and retry logic.
- Database:
  - Check migrations logs, schema versions, and transactional boundaries; replay failing migration on snapshot.
- DI:
  - Analyze missing binding logs, scopes, and component lifecycles.

Root Cause Analysis (RCA) Template
----------------------------------
Every RCA should contain:
- Problem summary (1–2 sentences)
- Impact: scope, affected versions, user sessions, crash rate delta
- Artifacts: list of logs, traces, repro, and commands used
- Timeline: discovery → diagnosis → fix
- Root cause (technical): detailed explanation with code pointers
- Evidence: stack traces, heap diffs, perf graphs supporting cause
- Fix implemented: diffs/files changed, tests added
- Risk & mitigation: regression risk and rollout plan
- Preventive measures: lint rules, tests, monitoring, docs
- Postmortem owner and date

Standard Tools & Integrations
-----------------------------
- Local & CI tools:
  - adb, adb logcat, dumpsys, dumpsys meminfo, dumpsys cpuinfo
  - adb bugreport, hprof-conv, heap dump tools
  - MockWebServer for network repro
  - Robolectric for JVM-level repros
  - LeakCanary for dev leak detection
  - Perfetto and systrace for performance traces
  - Android Studio Profiler
- Observability:
  - Crashlytics, Sentry, Play Console, Firebase Performance, Prometheus/Datadog integration
- Repository & CI:
  - Git blame / bisect to find regression commits
  - CI matrix across API levels, languages, locales, ABI
- Static analysis:
  - Lint, Detekt, ErrorProne, R8/ProGuard mapping checks

Automated Triage & Prioritization
--------------------------------
- Use crash grouping (stack fingerprint) to deduplicate incidents.
- Prioritize by user impact, crash-free rate impact, privacy/security, and regressions post-release.
- If stack trace maps to recent commit(s), assign to that owner with high priority.

Safe-fix Patterns (examples)
----------------------------
- NullPointerException:
  - Validate incoming data, add null checks, or use Kotlin nullable types + safe-calls.
  - Prefer defensive validation at API boundaries and add unit tests.
- IllegalStateException from lifecycle misuse:
  - Use lifecycleScope and viewLifecycleOwner for Fragment coroutines; avoid view references after onDestroyView.
- ANR due to long IO on main:
  - Move blocking IO to IO dispatcher or WorkManager; add timeouts and cancellation.
- Memory leak from listeners:
  - Unregister listeners in appropriate lifecycle callbacks; use weak references or lifecycle-aware listeners.
- Concurrent modification or race:
  - Use synchronized primitives, mutex, or single-threaded dispatcher; add concurrency tests.
- Retrofit serialization crash:
  - Add safe parsing, validate server contracts, and include fallback parsing with graceful error mapping.

Change & Testing Requirements
-----------------------------
- Every code fix must include:
  - Unit tests covering the failure and expected behavior.
  - Integration/test harness reproducer where possible (MockWebServer, emulator).
  - UI tests for user-facing fixes (Espresso/Compose).
  - Lint rules or static checks if the defect results from common anti-pattern.
- CI must run affected tests in a matrix of API levels and variants.
- Feature flags / staged rollout for high-risk fixes.

Monitoring, Verification & Rollout
----------------------------------
- Post-merge, run:
  - Canary release or staged rollout (Play Console).
  - Monitor crash rate & key metrics for at least one major release window.
  - If regression occurs, roll-back or hotfix with emergency patch.
- Add or update dashboards/alerts for the fixed metric (e.g., crash fingerprint, latency percentile).

Security, Privacy & Compliance
------------------------------
- Never include PII, user tokens, or sensitive payloads in logs or crash reports.
- Redact or hash identifiers before sending to telemetry.
- For GDPR / regional compliance, ensure logs and reports respect user opt-outs.

Preventive Measures & Hardening
-------------------------------
- Add runtime checks & assertions for invariants found in RCA.
- Add unit/integration tests for common failure modes.
- Add lint rules to prevent recurring anti-patterns.
- Increase observability in suspect areas: add structured logs, traces, metrics, correlation ids.
- Add chaos testing for resilience-critical components where appropriate (timeouts, network faults).

Templates & Examples
--------------------
- Quick stack trace analysis checklist:
  1. Identify topmost app frame in stack trace.
  2. Map to source via R8 mapping if obfuscated.
  3. Inspect surrounding code, nullability, lifecycle, or thread context.
  4. Reproduce or create small unit test reproducer.
  5. Fix + tests + CI.
- Example adb commands to collect artifacts:
  - adb logcat -d > logcat.txt
  - adb bugreport bug.zip
  - adb shell am dumpheap <pid> /sdcard/heap.hprof
  - adb pull /sdcard/heap.hprof .
  - hprof-conv heap.hprof heap-conv.hprof
- Perfetto capture:
  - Use Android Studio or command line: perfetto --config=myconfig.pb --out=trace.perfetto-trace
- Heap dump analysis:
  - Open converted .hprof in Android Studio Memory Analyzer, inspect dominator tree and GC roots.

Deliverables (per debugging task)
--------------------------------
- Problem summary + RCA document (use RCA template).
- Reproducer (automated test or steps).
- Minimal code fix with unit & integration tests.
- CI changes (test additions, lint rules) if needed.
- Observability updates (metrics/alerts/dashboards).
- Post-deploy monitoring plan and verification results.
- Postmortem with preventive steps and owner.

Pre-merge Checklist
-------------------
- [ ] Reproducible test or clear reproduction steps included.
- [ ] Fix limited to necessary files; design rationale documented.
- [ ] Unit & integration tests added and passing.
- [ ] Verified across impacted Android versions / ABI / locales.
- [ ] No sensitive data added to logs; telemetry validated.
- [ ] Rollout plan / feature flag considered for high-risk changes.
- [ ] RCA document attached to PR and incident tracker.

Governance & Ownership
----------------------
- Debugging Skill is owned by Debugging Team / on-call owners.
- High-severity incidents require postmortem within agreed SLA.
- Changes to debugging processes or templates require cross-team review and update of docs/CHANGELOG.

Appendices
----------
A. Example RCA structure (boilerplate)
   - Title:
   - Incident ID:
   - Reported by:
   - Dates & timeline:
   - Summary:
   - Impact:
   - Artifacts:
   - Root cause:
   - Fix:
   - Tests:
   - Post-deploy verification:
   - Lessons learned:
   - Action items & owners:

B. Quick reference: common root causes
   - Null access after view destroyed -> Fragment view lifecycle misuse
   - Blocking main thread -> disk or network IO on main
   - Leak via static reference -> context leak
   - ANR from BroadcastReceiver -> long work on main
   - Crash after Proguard -> missing keep rules for reflection

C. Common adb / diagnostic snippets
   - adb logcat -v threadtime > crash-window.log
   - adb shell setprop debug.firebase.analytics.app com.example.myapp (for debug)
   - dumpsys package <package> > package-info.txt

D. Recommended dev tooling
   - LeakCanary in debug builds
   - Perfetto + Android Studio Profiler
   - MockWebServer fixtures for network testing
   - Local crash reproduction harness tied to unit tests
   - CI test matrix for API levels in use by production

E. Example small change policy
   - For urgent high-impact fixes: small PRs limited to 1–3 files are acceptable for emergency deploys, but must be followed by full review and expansion of tests within 48 hours.

Contact & Next steps
--------------------
- Add this file at docs/debugging-skill.md.
- Integrate available templates into incident management and CI pipelines.
- Optionally I can scaffold:
  - RCA template file and PR checklist workflow.
  - Sample reproducer harness (MockWebServer + unit test) for a given crash.
  - CI job that runs perf/trace reproducers for regression detection.
