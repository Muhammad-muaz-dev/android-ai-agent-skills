---
name: permissions
description: Handle Android runtime permissions, scoped storage, media access, notifications, foreground services, rationale UI, denial states, and policy-safe permission flows.
---

# Permission Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Permissions Team / permission-skill agent

Purpose
-------
This Permission Skill defines a scalable, maintainable, lifecycle-aware, and policy-compliant permission management architecture for Android apps. It standardizes how permissions are discovered, requested, rationalized, persisted, and recovered across platform versions (Android 10–15 and beyond). The skill strictly confines permission logic to a centralized permission layer and prohibits embedding permission handling into business logic or unrelated UI code.

Scope
-----
Covers:
- Permission discovery and mapping per feature
- Centralized permission manager API (requesting, checking, observing)
- Lifecycle-aware request flows for Activities/Fragments and Compose
- Platform-specific behavior (scoped storage, photo picker, background location, notifications, Bluetooth, exact alarms, special permissions)
- Rationale and educational flows, deep linking to app settings when needed
- One-time permissions, limited access, and auto-reset handling
- Testing, linting, CI checks, and analytics/telemetry guidance (privacy-first)
- Manufacturer and device-specific fallbacks and mitigation

Excludes:
- Business logic decisions based on permission outcomes (beyond enabling/disabling feature entry)
- UI design beyond recommended patterns (specific UX tailored by designers)
- Network/database implementations

High-level Principles
---------------------
1. Principle of Least Privilege — request only the minimal permission required, and only when the user initiates the relevant action.
2. Single source of permission truth — a centralized PermissionManager (and typed contracts) is the canonical API for all permission interactions.
3. UI-driven requests — permission prompts must originate from UI layer (Activity/Fragment/Composable). PermissionManager provides lifecycle-aware helpers, not direct calls from ViewModel/business logic.
4. Progressive disclosure — show rationale before request when appropriate; avoid surprise prompts.
5. Platform-aware — differentiate behavior by Android version and follow the latest platform guidance (Photo Picker, scoped storage, foreground services, notification runtime permissions, etc.).
6. Testable & injectable — make permission behaviors mockable/fakeable for unit & integration tests.
7. Privacy-first telemetry — any telemetry generated about permission usage must not contain PII and must follow consent rules.

Responsibilities (what this Skill must deliver)
-----------------------------------------------
- Identify required permissions per feature and maintain a permission matrix.
- Provide a centralized PermissionManager API with lifecycle-aware helpers (Activity/Fragment/Compose integration).
- Implement rationale and fallback flows, including "permanent denial" recovery (open settings).
- Handle Android version-specific differences and special permissions robustly.
- Provide one-time permission handling and support partial media access (Photo Picker).
- Offer typed results and event streams for permission state changes (Flow).
- Provide unit/integration test fakes, emulator test instructions, and CI lint rules.
- Document permission decisions, UX rationale, and Play Store justification text for reviewed permissions.

Core Rules (enforced)
---------------------
- Request permissions only when the user triggers an action that needs them.
- Do not request multiple unrelated permissions at once without a strong UX justification.
- Keep permission logic out of ViewModels and business layers — only UI controllers and the centralized PermissionManager interact with platform APIs.
- Always present rationale when a previously denied permission requires re-request explaining the benefit.
- Use one-time permissions and Photo Picker where possible to avoid broad storage access.
- Use lifecycle-aware constructs to avoid leaks and invalid callbacks (e.g., use Activity/Fragment result APIs, lifecycleScope, or rememberLauncherForActivityResult in Compose).
- Provide clear, privacy-preserving logs and telemetry. Never log PII or raw permission tokens.
- Ensure compliance with Google Play policies; prepare Play listing justification text for every sensitive permission.

Permission Types & Platform Awareness
------------------------------------
Support and handle differences for:
- Normal & Dangerous permissions (e.g., CAMERA, RECORD_AUDIO, ACTIVITY_RECOGNITION)
- Location (foreground vs background; request foreground first, then background with separate justification)
- Notifications (Android 13+ runtime permission POST_NOTIFICATIONS)
- Media: Photo Picker, READ_MEDIA_IMAGES/VIDEO/AUDIO on Android 13+, scoped storage (Android 10+)
- Bluetooth & Nearby Devices (Android 12+ Bluetooth permissions)
- Foreground services, exact alarms, and foreground-service types
- Device admin and special permissions (SYSTEM_ALERT_WINDOW, MANAGE_EXTERNAL_STORAGE, REQUEST_INSTALL_PACKAGES, WRITE_SETTINGS, ACCESS_NOTIFICATION_POLICY)
- Manufacturer-specific behavior (MIUI, Huawei, Samsung) — document known issues and graceful fallback.

Permission State Model
----------------------
Centralize permission states with explicit types:

- Granted
- Denied (temporary)
- PermanentlyDenied (user selected "Don't ask again" or OS equivalent)
- Limited (e.g., photo picker gives limited media access)
- OneTimeGranted (granted for one-time use where OS supports)
- Revoked (by system or owner)
- Unsupported (permission not applicable to OS version/device)

PermissionManager API (conceptual)
----------------------------------
Design a single PermissionManager abstraction consumers use:

- suspend fun check(permission: Permission): PermissionState
- fun observe(permission: Permission): Flow<PermissionState>
- fun requestPermission(activityOrFragment, permission: Permission, rationale: Rationale?): Flow<PermissionResult>
- fun requestPermissions(activityOrFragment, permissions: List<Permission>, rationale: Rationale?): Flow<MultiPermissionResult>
- fun openAppSettings(activityOrFragment): Unit

Types/Models:
- Permission: typed enum or sealed class representing permissions and platform mapping
- Rationale: { title: String, message: String, positiveText: String, negativeText: String, escalateToSettings: Boolean }
- PermissionState (see states above)
- PermissionResult: Success | Denied | PermanentlyDenied | Canceled
- MultiPermissionResult: Map<Permission, PermissionResult> plus aggregate result helpers

Design patterns:
- UI controllers call PermissionManager.requestPermission(...) and subscribe to returned Flow to receive the result.
- PermissionManager internally uses AndroidX Activity Result APIs or Compose Launchers and maps responses to typed PermissionResult.
- Expose observe(...) for features that need continuous monitoring of permission state (e.g., location tracking).

Rationale & UX Guidelines
-------------------------
- Rationale must be shown before requesting when:
  - The user previously denied the permission.
  - The permission is sensitive (location background, microphone, camera).
- Rationale must:
  - Explain why permission is needed and how it benefits the user.
  - Offer "Continue" (trigger request) and "Cancel" (do not request) actions.
  - For background or special permissions, include escalation steps and explain how to open settings.
- Use inline contextual help where possible instead of blocking modal dialogues.
- If permission is denied permanently, guide users with an "Open Settings" flow and show consequences.

One-time Permissions & Photo Picker
-----------------------------------
- Prefer the platform Photo Picker (Android 13+ and AndroidX equivalents) to avoid broad storage permissions.
- If the app requires persistent file access across sessions, use the Storage Access Framework or request scoped storage permissions with clear justification.
- Implement specific code paths for limited access (READ_MEDIA_* on Android 13+) and gracefully degrade behavior when limited.

Background Location
-------------------
- Only request background location after the app has a strong foreground usage signal and rationale dialog.
- Follow Play Store policies: provide in-app geolocation justification and Play Console declaration when requesting background location.
- Request foreground location first; background location is a separate request and must be requested in a distinct flow.

Special Permissions & System Settings
-------------------------------------
- Only request special permissions (SYSTEM_ALERT_WINDOW, MANAGE_EXTERNAL_STORAGE, WRITE_SETTINGS, SCHEDULE_EXACT_ALARM, REQUEST_INSTALL_PACKAGES) after presenting a detailed explanation and second-confirmation UI.
- For MANAGE_EXTERNAL_STORAGE, prefer MediaStore and Photo Picker; avoid MANAGE_EXTERNAL_STORAGE unless absolutely necessary and justified to Play Store.
- Use dedicated flows to open system settings for these permissions; never try to bypass system UI.

Debounce & Reentrancy
---------------------
- Prevent rapid repeated permission requests: implement UI-level debouncing or PermissionManager guards to avoid spamming the system prompt.
- Avoid issuing permission prompts while the app is backgrounded or during configuration changes. Use lifecycle-aware APIs and queue up requests until UI is visible.

Fragment/Activity/Compose Integration Patterns
----------------------------------------------
- Activities/Fragments: use ActivityResultContracts.RequestPermission and RequestMultiplePermissions, wired through PermissionManager.
- Compose: expose rememberPermissionLauncher wrappers that call PermissionManager under the hood.
- Ensure request flows use viewLifecycleOwner.lifecycleScope or lifecycleScope to avoid leaks.
- Use distinct request codes/identifiers to map responses when multiple concurrent requests are possible.

Testing & CI
------------
- Unit tests:
  - Provide a FakePermissionManager for unit tests to simulate states: granted, denied, permanentlyDenied, limited.
  - Test business logic with fake manager injected.
- Integration tests:
  - Use UI tests (Espresso/ComposeTestRule) to assert rationale displays, open settings flow, and correct feature enabling/disabling.
  - On emulators, toggle permission states via adb where possible to simulate user responses.
- Lint & CI checks:
  - Lint rule to detect permission requests originating from ViewModels (fail).
  - Lint rule to enforce rationale presence before background-location or other sensitive permission requests.
  - CI to run UI flows and check Play policy compliance checklist (e.g., background location declaration).
- Provide test fixtures that exercise platform differences (Android 10–15+) in CI matrix.

Telemetry & Analytics (Privacy-first)
------------------------------------
- Track aggregated metrics: permission request counts, grant rates, time-to-grant, common refusal reasons (if explicitly provided), and settings opens.
- Do not log or send PII, exact content, or sensitive tokens.
- Respect user consent for analytics; if analytics are disabled, do not emit telemetry.
- Use rate-limited, privacy-preserving telemetry for diagnosing UX issues.

Device & Manufacturer Quirks
----------------------------
- Document known issues (e.g., MIUI aggressive permission managers) and provide mitigation steps:
  - Detect problematic OEM and show vendor-specific guidance if needed.
  - Provide "Open vendor settings" instructions where possible.
- Always fallback gracefully; do not fail feature-critical paths due to vendor-specific behavior.

Permission Audit & Play Store Compliance
---------------------------------------
- Maintain a permission matrix mapping features → required permissions → Play Store justification text.
- Keep sample phrases for Play Console permission declaration.
- For sensitive permissions, maintain documentation explaining why the permission is required and how user privacy is preserved.

Migration & Versioning
----------------------
- Version the Permission Skill docs using semver.
- Track permission contract changes (e.g., adding a new required permission) in a CHANGELOG and require cross-team sign-off.
- Provide migration guides for breaking permission flows (e.g., when moving to Photo Picker or scoped storage changes).

Deliverables (per feature/implementation)
----------------------------------------
- Permission analysis (matrix: feature → permissions → Android versions).
- PermissionManager interface and integration shims for Activity/Fragment/Compose.
- Rationale copy and UX flow descriptions.
- Unit test coverage & FakePermissionManager.
- Integration/UI tests (including adb-driven permission toggles for emulators).
- Play Store justification text and privacy impact statement.
- Migration notes (if changing permission contracts).
- Lint rules added or documented.
- Example code snippets and templates.

Pre-merge Checklist
-------------------
- [ ] PermissionManager API implemented and injected via DI.
- [ ] No direct permission requests from ViewModels or non-UI layers.
- [ ] Rationale dialogs exist for sensitive or repeat-denied permissions.
- [ ] One-time and limited-media flows implemented where applicable.
- [ ] Background location and other sensitive permissions follow separate request flows and Play policy checklists.
- [ ] Special permissions are justified, documented, and gated behind explicit user flows.
- [ ] Unit tests using FakePermissionManager are present.
- [ ] Integration/UI tests for common permission flows exist.
- [ ] Lint rules prevent incorrect patterns (requests from non-UI, missing rationale).
- [ ] Play Console permission justification text prepared and recorded.
- [ ] Telemetry follows privacy rules and honors user consent.

Examples & Implementation Sketches
---------------------------------

Types & models (Kotlin):
```kotlin
// permission/models/PermissionTypes.kt
sealed class AppPermission(val androidName: String) {
  object Camera : AppPermission(Manifest.permission.CAMERA)
  object RecordAudio : AppPermission(Manifest.permission.RECORD_AUDIO)
  object FineLocation : AppPermission(Manifest.permission.ACCESS_FINE_LOCATION)
  object BackgroundLocation : AppPermission(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
  // Add other permissions with mapping for platform differences
}

// Permission state/result types
enum class PermissionState { GRANTED, DENIED, PERMANENTLY_DENIED, LIMITED, ONE_TIME, UNSUPPORTED }

sealed class PermissionResult {
  object Granted : PermissionResult()
  data class Denied(val shouldShowRationale: Boolean) : PermissionResult()
  object PermanentlyDenied : PermissionResult()
  object Canceled : PermissionResult()
}
