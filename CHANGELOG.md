# @lam/vulture/std-plugin Changelog

All notable changes to `@lam/vulture/std-plugin` are documented here.

## 2.0.0 - 2026-09-30

### Added

- Configurable `*Plugin(...) -> func(Server)` factories for use with Vulture
  v2's `Server.register`; parameterless installers remain directly registerable.
- Typed Vulture v2 test coverage using `InjectResponse`, `RouteOptions`, typed
  lifecycle APIs, and plugin registration.

### Changed

- Canonicalized documentation-comment delimiters and repaired HTTP terminology
  and compound-word spacing in plugin API prose.
- **Breaking:** renamed `CircuitBreaker.callFn` to `CircuitBreaker.call`. It
  accepts a typed zero-argument callback and retains the existing `Result`
  contract (`CircuitOpen` and `CircuitFailed` errors, `Ok(None)` on success).
- **Breaking:** migrated every class constructor from `__init__` to the Lam 1.18
  `init` convention.
- Migrated plugin installers from direct runtime slice and route mutation to
  Vulture v2's typed handler, hook, route, decorator, and plugin APIs.
- Typed the bearer verifier as `func(str) -> any`; the return remains
  intentionally heterogeneous so applications can store arbitrary claims.
- Moved request logging and server timing orchestration into typed Lam hooks and
  kept cache-control policy entirely in Lam. Existing security-header and
  cache-header precedence behavior is unchanged.
- Replaced legacy dictionary injection results and option helpers in tests with
  typed `InjectResponse` and `RouteOptions` APIs.
- Raised compatibility to Lam 1.18, updated `@lam/vulture` to v2.0.0, and bumped
  this package to v2.0.0.

### Compiler boundary

- Vulture request context is intentionally heterogeneous and `Context.get`
  returns `any`. Lam 1.18 cannot language-narrow that value, so request logging
  and server timing use two private, checked conversion helpers. Native plugin
  state remains private `any`; public executable extension points are typed.

### Testing

- Added installed-package composition coverage for auth, cache control, tracing,
  plugin factories, fluent HTTP calls, and circuit-breaker recovery.

## 1.0.0 - 2026-09-18

### Added

- Initial standalone standard-plugin release for `@lam/vulture` v1.
