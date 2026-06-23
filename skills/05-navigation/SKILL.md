---
name: navigation
description: Design and implement Android navigation flows with Jetpack Navigation, deep links, back stack behavior, arguments, nested graphs, and safe screen transitions.
---

# Navigation Skill — Production-grade Spec
Version: 1.0.0
Last updated: 2026-06-20
Owner: Navigation Team / navigation-skill agent

## Purpose
This Navigation Skill defines a production-ready, scalable, and maintainable navigation architecture using the Android Navigation Component. It prescribes conventions, organization, and deliverables so any agent or engineer can implement consistent navigation across the application without adding business logic or data-layer concerns.

This document is the single source of truth for navigation architecture, integration rules, back stack policies, deep-link behavior, state-restoration, testing guidance, and versioning.

## Scope
This skill covers:
- Navigation architecture design and implementation using the Android Navigation Component.
- Graph organization and modularization (feature graphs, nested graphs).
- Back stack and Up behavior.
- Safe Args (type-safe navigation).
- Deep links (internal and external).
- Authentication and onboarding flows as navigation responsibilities only.
- Dialog and BottomSheet destinations.
- State restoration and process-death handling.
- Integration guidelines for other agents/features.
It explicitly excludes business logic, networking, database, analytics, and non-navigation UI behavior.

## High-Level Principles
1. Single Activity Architecture (default)
   - Use a single Activity as the host (e.g., MainActivity) unless multiple activities are explicitly justified (task-level isolation, system-level intents).
2. Navigation Component is the canonical API
   - All navigation is done through Navigation Component APIs (NavController, NavHostFragment, NavGraph, NavigationUI) and Safe Args for arguments.
3. Separation of concerns
   - Navigation actions originate only from the UI layer (Activity/Fragment/View).
   - ViewModels must be independent of navigation implementations (no NavController/Context leaks).
4. Modularity & Scalability
   - Feature teams own feature NavGraphs; root graph composes features.
   - Support Dynamic Feature Modules via nav graph modularization and runtime graph inflation.
5. Predictability & Testability
   - Define consistent back stack semantics, global actions, and navigation edge cases; ensure flows are unit and integration tested.

## Responsibilities (Agent must)
- Design navigation graphs & destination hierarchy.
- Implement navigation with Navigation Component and Safe Args.
- Provide nav-graph templates and examples for feature modules.
- Configure Bottom Navigation and Navigation Drawer navigation integration.
- Implement nested nav graphs, dialog & bottom sheet destinations.
- Implement deep links and argument validation.
- Provide state restoration strategies for configuration changes and process death.
- Define back stack semantics and logout flows that clear sensitive state.
- Deliver test cases, lint rules, and CI checks for navigation correctness.
- Document migration/versioning for graph changes.

## Rules (enforced)
- Use Navigation Component (no custom navigation middleware unless justified and documented).
- Keep activities minimal (host + deep-link entry points only).
- Fragments are the primary destinations; Activities only if necessary (e.g., full-screen single-purpose host, external task).
- Use Safe Args for all typed arguments.
- Navigation logic must not contain business logic: perform only destination decisions and argument validation.
- ViewModels must not depend on Android navigation APIs.
- Avoid invoking navigation more than once per user action (debounce clicks at UI layer or use single-consumption events).
- Use global actions only for navigation that originates from multiple graphs.
- All navigation graph changes must be backward compatible or follow a documented migration path.

## Navigation Types Supported
- Fragment Navigation (primary)
- Activity Navigation (when necessary)
- Bottom Navigation / Top-level tabs
- Navigation Drawer
- Nested Navigation Graphs
- Deep Links (internal & external)
- Global Actions
- Authentication & Onboarding Flows
- Dialog & Bottom Sheet Destinations
- Modal Navigation & Conditional flows
- Dynamic Feature Module Navigation
- External Intent Navigation (implicit intents)

## Graph Organization & Conventions
- Root graph (app-level): /navigation/nav_graph_app.xml
  - Hosts top-level graphs (home_graph, auth_graph, settings_graph, feature graphs).
- Feature graph per feature/module:
  - Conventions: res/navigation/feature_<name>_graph.xml inside that module.
- Naming:
  - Graph files: feature_<feature-name>_graph.xml
  - Destinations: fragment_<Feature>_<Purpose> e.g., fragment_profile_edit
  - Actions: action_<from>_to_<to> e.g., action_home_to_profile
  - Argument names: use snake_case or lowerCamel consistently across the repo.
- Each feature exposes:
  - nav-graph resource
  - optional public deep-link URI prefix (documented in module README)
  - a Navigation module entry point (if using dynamic features)
- Keep graphs small and focused: each graph should represent a user flow or feature.

## Single Activity & NavHost setup
- Single Activity hosts a single NavHostFragment (or multiple named NavHostFragments for special cases such as nested modal hosts).
- NavHostFragment should be created in layout XML and wired using FragmentContainerView + NavHostFragment.
- Access NavController via:
  - val navController = findNavController(R.id.nav_host_fragment)
  - For nested fragments, use findNavController() on view.

## Safe Args (Type-Safe Navigation)
- Mandatory for argument passing.
- Gradle setup:
  - Apply Safe Args plugin in all modules that define or consume navigation graphs.
- Argument validation:
  - All arguments must declare type, default value (if optional), and navArgs must be validated before use.
- Avoid passing complex domain objects; prefer IDs and small serializable DTOs. Use ViewModel+repository to fetch large objects.

## Back Stack Strategy
- Define top-level destinations (tabs) to be replaceable without creating duplicates:
  - Use popUpTo with inclusive=false for switching tabs to maintain local back stacks per tab if desired.
- Prevent duplicate destinations:
  - Use launchSingleTop for actions to top-level destinations.
- Logout / sensitive flows:
  - Use popUpTo on logout with inclusive=true and set new graph or navigate to auth_graph clearing back stack.
- Up navigation:
  - Up should respect nested graphs. When at root of nested graph, Up should route to parent graph default destination.
- Clear navigation state for privacy:
  - On logout or account switch, clear back stack and any saved nav state that references sensitive routes.

## Nested Graphs & Modularity
- Feature graphs are embedded as <include app:graph="@navigation/feature_graph" /> or added programmatically for dynamic features.
- Parent graphs own global actions and coordinate cross-feature navigation.
- Minimize cross-graph tight coupling: prefer communicating via navigation arguments or well-defined global actions.

## Bottom Navigation & Multi-backstack
- For BottomNavigationView, store each tab's NavController back stack state (using saveState/restoreState on NavController or keep one NavHost per tab pattern).
- Implementation options:
  - Single NavHost + multiple top-level destinations (recommended for simplicity) OR
  - Multiple NavHostFragments (one per tab) to preserve independent back stacks.
- Keep tab selection state in savedStateHandle or in UI layer; never in ViewModel as navigation state.

## Deep Links
- Support both internal and external deep links.
- Register deep links on destinations (app:navGraph or destination-level deepLink).
- Validate arguments for deep links and handle invalid / malicious data gracefully:
  - If required args are missing/invalid, redirect to safe fallback (e.g., home or auth).
  - If destination requires authentication, redirect to auth flow with a pending-deep-link token that resumes after login.
- External deep links:
  - Use an explicit mapping table (host/path -> destination) documented in module-level README.
  - Sanitize and validate URIs before navigation.
- Security:
  - Never perform sensitive operations solely based on external deep link without authentication checks.

## Authentication & Conditional Flows
- Navigation handles routing decisions, not authentication logic.
- Patterns:
  - Guard destination with a navigation interceptor in Activity/Fragment that checks authentication and performs redirect when needed.
  - Use a navigation graph for auth (auth_graph) with global action to return to the original destination after auth success.
- When redirecting to auth, capture the intended destination (serialized arguments or deep link) using SavedStateHandle or a token; resume navigation on success.

## Onboarding Flow
- Onboarding is a separate graph (onboarding_graph); it can be launched conditionally at first run.
- Ensure onboarding is idempotent and supports resume after process death.
- After onboarding, navigate to the appropriate starting point and clear onboarding from back stack.

## Dialog & BottomSheet Destinations
- Use dialog destinations defined in navigation graphs (dialog/destination) for consistent lifecycle and back handling.
- Prefer BottomSheetDialogFragment for bottom-sheet destinations; register as dialog destination in graph.
- Dialogs should not own navigation logic beyond dismissing and returning results via savedStateHandle or ViewModel.

## State Restoration & Process Death
- Use Navigation Component's state saving and SavedStateHandle for ViewModels.
- Persist selected tabs and current navigation destination in SavedStateRegistry or use NavController.saveState() & restoreState() where appropriate.
- On process death:
  - Provide a recovery path: valid deep link or landing page (home or auth) and rehydrate ViewModels by IDs passed in nav args.
- Avoid storing large UI state in navigation arguments.

## Error Handling & Robustness
- Prevent invalid navigation:
  - Validate arguments before calling navController.navigate().
  - Use try/catch around deep link parsing and fallback to safe route.
- Debounce navigation to avoid duplicate clicks (UI layer responsibility).
- Fail gracefully:
  - If a destination is missing (e.g., feature module not installed), show a placeholder or prompt to install module, and avoid crash.
- Logging:
  - Navigation agent may emit structured logs for failures to help debugging, but must avoid sensitive data.

## Dynamic Feature Modules
- Feature modules declare their own navigation graph resources.
- Register graph lazily:
  - When a graph requires a dynamic feature that is not installed, show a friendly UI and request installation. Upon install, inflate and add the graph then navigate.
- Ensure deep links to dynamic features trigger install flow and resume navigation on success.

## Testing & CI
- Unit tests:
  - Test navigation destinations and actions using NavController testing helpers (TestNavHostController).
  - Validate argument types and default values.
- Integration tests:
  - Test major flows (auth->home, onboarding, deep-link resume) using instrumentation tests.
- UI tests:
  - Espresso tests that exercise bottom nav, nav drawer, and back stack behavior.
- CI checks:
  - Add lint rules to detect direct NavController usage in ViewModels, missing Safe Args, and deprecated nav graph references.
  - Fail builds if navigation graphs use hard-coded business logic patterns.
- Provide sample tests in each feature module as templates.

## API & Integration Contract for Other Agents
- Navigation Skill exposes a minimal contract to feature agents:
  - Public nav graph resource path e.g., navigation/feature_x_graph.xml
  - Optional navigation helper object for deep link formation (pure string templating, no business logic).
  - Documented list of arguments and expected types for public entry points.
- Agents must:
  - Not modify navigation spec without coordination.
  - Use Safe Args and documented actions only.
  - Open issues/PRs for new navigation flows that cross module boundaries.

## Versioning & Migration
- Semantic versioning for the navigation skill spec (major.minor.patch).
- Migration policy:
  - Backward-compatible graph changes allowed (add destinations, non-breaking args).
  - Breaking changes (remove action/destination or change arg type/name) require migration plan and feature deprecation period.
  - Provide a migration checklist and code mod recipes (search/replace patterns).
- Keep changelog in the repo (docs/navigation/CHANGELOG.md).

## Deliverables (every navigation implementation)
- Navigation graph(s) with documentation and per-destination descriptions.
- Destination hierarchy diagram (SVG/PNG) or ASCII map.
- Back stack strategy description for the flow.
- Deep link table (URI -> destination).
- Safe Args models and description of each argument (type, required/optional, validation).
- Integration notes for the feature module (public entry points, graph id).
- Test suite (unit + integration + UI).
- Lint rules and CI checks (or links to implemented checks).
- Migration notes if applicable.

## Checklist (pre-merge)
- [ ] Navigation uses Navigation Component, NavGraph and Safe Args.
- [ ] Graph is modular and included in root graph (or documented reason for standalone).
- [ ] Arguments typed via Safe Args; no large domain objects passed.
- [ ] Deep links documented and sanitization present.
- [ ] Back stack behavior described and validated via tests.
- [ ] Authentication / Onboarding sequences documented and resumed correctly.
- [ ] Dialogs / BottomSheets declared as dialog destinations.
- [ ] State restoration strategy documented and tested.
- [ ] No business logic in navigation; business actions delegated to ViewModel or interactor.
- [ ] CI includes navigation lint/tests

## Example: Minimal Feature Graph (reference)
- File: res/navigation/feature_profile_graph.xml
- Key points:
  - Defines only profile-specific destinations.
  - Exposes one public deep link for external entry.
  - Relies on Safe Args for profileId.

(Actual XML template and Safe Args classes are available on request. Keep XML templates in /templates/navigation/ for the repo.)

## Implementation Guidelines (short)
- Always call navController.navigate(action) from UI controllers (Fragments/Activities).
- Use viewLifecycleOwner.lifecycleScope for navigation triggered by coroutine callbacks to avoid lifecycle races.
- Use event wrappers or single LiveData events to prevent repeat navigation from configuration changes.
- Use NavOptions (popUpTo, launchSingleTop) to control back stack transitions intentionally.
- For cross-module navigation, prefer global actions or use deep links as the contract.

## Linting Rules (recommendations)
- Fail if:
  - Safe Args not used when arguments declared.
  - NavController is injected into a ViewModel.
  - Fragments perform long-running work during navigation (should use ViewModel).
- Warn if:
  - Graph size > X destinations (suggest split).
  - Dynamic features referenced without install check.

## Security & Privacy
- Never store sensitive tokens in navigation arguments or nav state.
- Sensitive destinations must require re-authentication if appropriate.
- Deep links must be validated and sanitized.

## Governance & Ownership
- Navigation Skill is owned by the Navigation Team (or owner alias).
- Any change requires a PR with:
  - Graph diff
  - Migration plan for breaking changes
  - Updated tests
  - Updated diagrams and documentation

---

Add this file to your repository (recommended path: docs/navigation-skill.md). If you want, I can:
- Generate XML nav graph templates for root and a feature module following these conventions.
- Add Gradle Safe Args plugin snippets and a sample MainActivity/Host Fragment implementation.
- Create sample tests for one feature graph.

If you'd like the concrete nav graph XML templates and Kotlin helpers, tell me which module or feature you want first and I will generate them next.
