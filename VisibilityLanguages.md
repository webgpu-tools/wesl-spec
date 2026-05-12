Survey of visibility, re-export, and wildcard handling.
[Visibility.md](Visibility.md) and
[VisibilityDesign.md](VisibilityDesign.md).

_this is background for reviewers, not a published part of the spec, remove before merging_

Carbon
- 3 levels at present: library private, file private, public
- uses a separate api file
- cross-package re-export is banned
- no wildcards

Go
- 2 levels
- the directory is the package boundary (subdirectories are separate packages)
- partial 3rd level: `internal/` directories scope packages to a subtree
- first letter determines visibility; convention is private by default
- no re-exports
- has wildcards (`import . "pkg"`), strongly discouraged except for tests

Java
- 4 levels: public, protected (= package + subclasses), package-private
  (default), private
- JPMS adds module-level visibility; `requires transitive` propagates modules
  only, not items
- no item-level re-exports
- `@VisibleForTesting` (Guava, AndroidX) marks members whose visibility has been
  widened to enable tests
- wildcards, but forbidden by e.g. Google style guide

Kotlin
- 4 levels
- `internal` means compilation unit
- public by default
- no re-exports
- no named friends/super/etc.
- wildcards, but largely discouraged (some tools auto-expand)

Rust
- 5 levels (also `pub(in crate::a::b)`, `pub(super)`)
- private by default
- re-exporting allowed internally and externally; illegal to re-export
  package-visible as public-visible
- wildcards, some style guides discourage

Scala
- 4 levels (`private[pkg]`, `protected[pkg]`)
- visibility control is also hierarchical: `private[pkg.foo]` widens access to
  anything inside `pkg.foo`, including all subpackages (it can only widen, never
  narrow, plain `private`)
- public by default
- re-exporting, also allows filtering `export printer.{status as _, *}`;
- wildcard imports allowed (also with filtering)

Slang
- 3 levels: public, private, and `internal`
- `internal` is per module (no package-level visibility control)
- modern modules default to `internal`; legacy modules remain public by default
  for compatibility
- re-export via `__exported` (underscore-prefixed, not formally part of the
  language)
- module design is still evolving in Slang, e.g.
  [issue 9183](https://github.com/shader-slang/slang/issues/9183)
- wildcards by default: `import` flattens into the common namespace; authors can
  use `namespace`. (I wonder if this design comes from compatibility goals with
  HLSL `#include`.)

Swift
- six visibility levels: `open`, `public`, `package`, `internal`, `fileprivate`,
  `private` (the open/public split governs subclass and override permission;
  Swift's `package` means within a Swift Package Manager package (a manifest
  grouping several Swift modules); Swift's `internal` means within a single
  Swift module, where a Swift "module" is one compile target / library, closer
  in role to a WESL package than to a WESL module)
- default is Swift's `internal` (within one Swift module / library), which is
  roughly WESL's `package` default; (Swift's `package` is a distinct, wider
  level, not the default)
- re-exporting is supported via `public import` and via the unstable
  `@_exported import`
- imports flatten into the local namespace by default

Typescript
- two levels for declarations (+ levels for classes)
- `package.json` `exports` field for package-level visibility; combined with a
  barrel file (the `exports` entry point), gives a full 3rd level
- private by default
- re-exporting is supported
- wildcard syntax imports are not flattening (not really wildcards in our sense)

Zig
- two levels
- private by default (`pub` exports)
- re-exporting is allowed
- `usingnamespace` (the wildcard-ish mechanism) is on the way out

## Summary

| Language | Levels | Default | Package-level analog | Re-exports | Wildcards |
| --- | --- | --- | --- | --- | --- |
| Carbon | 3: file-private, library-private, public | api file public; impl file library/file-private | library-private | banned cross-package | none |
| C# | 6: `public`, `internal`, `protected internal`, `private protected`, `protected`, `private` | `internal` (top-level), `private` (nested) | `internal` | none | (not surveyed) |
| C++ | `public`, `protected`, `private` (class-level); modules orthogonal | `private` (class) | none (no module visibility pre-C++20) | (not surveyed) | (not surveyed) |
| Go | 2: exported (capital), unexported | unexported (by convention) | none (package is the unit) | none | `import . "pkg"` (test-only) |
| Java | 4: `public`, `protected` (package + subclasses), package-private, `private` | package-private | package-private (no keyword) | item-level: none; modules via `requires transitive` | yes (style guides discourage) |
| Kotlin | 4: `public`, `internal`, `protected`, `private` | `public` | `internal` (= compilation unit) | none | yes (discouraged) |
| Rust | 5: `pub`, `pub(crate)`, `pub(super)`, `pub(in path)`, private | private | `pub(crate)` | yes, internally and externally (no widening) | yes (some style guides discourage) |
| Scala 3 | `private`, `private[X]`, `protected`, `protected[X]`, public | public | `private[pkg]` | yes (with filtering); no wildcard re-export of packages | yes (with filtering) |
| Slang | 3: public, `internal`, private | `internal` (modern modules); public (legacy) | `internal` (per module) | `__exported` (underscore-prefixed) | imports flatten by default |
| Swift | 6: `open`, `public`, `package`, `internal`, `fileprivate`, `private` | `internal` | `package` (added in 5.9) | `public import`; `@_exported import` (flattening, unstable) | imports flatten by default |
| TypeScript | 2-3: exported, unexported (+ `package.json` `exports`) | unexported | none (module is the unit; `exports` field is the closest) | yes | namespaced under an identifier (never flattened) |
| WGSL | none | always module-public at module scope | none | n/a | n/a |
| Zig | 2: `pub`, private | private | none | yes | `usingnamespace` (on the way out) |
| **WESL** | **3: `public`, *package*, `private`** | ***package*** | ***package* (default, no keyword)** | **`public import` (public items only, no widening)** | **`import path::*` (flattens top-level decls); external needs `@wildcardable`** |
