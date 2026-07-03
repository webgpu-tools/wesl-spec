# Imports Design

This document records design decisions behind WESL's import system. The
normative spec lives in [Imports.md](Imports.md).

## Why the `@!` module attribute form?

`@!wildcardable` is module-level metadata. WESL will likely want module-level
annotations for other module-scoped features, and libraries and users will want
a place to attach their own metadata to a whole module. A general-purpose syntax
for module metadata avoids inventing one ad-hoc form per feature.

`@!wildcardable` mirrors the item-level attribute convention (`@group`,
`@binding`, `@if`, `@diagnostic`, ...), but the `!` marks the attribute as
scoped to the whole module. Unlike an item attribute, it doesn't attach to a
following element, so a trailing `;` terminates it.

Module attributes sit below the imports so that they can use imported
names (for example a hypothetical `@!play_version(2);`).

See [`@!wildcardable` annotation](Imports.md#wildcardable-annotation) for the
normative spec.

## Aren't wildcards an anti-pattern?

Many language communities discourage wildcard imports. TypeScript, Go, Zig, and
Carbon disallow wildcards entirely or restrict them to narrow cases. Java and
Rust permit them syntactically but discourage broad use by convention; Rust's
`prelude` modules are one curated pattern in that style. The general concerns
are practical:

- **Traceability.** Direct imports make it obvious where a name comes from.
  Wildcards push that work onto the reader, the language server, or the
  compiler's name-resolution diagnostics.
- **API stability.** Adding a public item to a wildcard-imported module can
  conflict with downstream declarations or with other wildcard imports. Stacked
  wildcards across a dependency tree can create conflicts the end user neither
  caused nor can easily fix.

The WESL environment adds further concerns:

- **Cross-ecosystem publishing.** WESL libraries can be published into multiple
  host ecosystems (npm, crates, etc.), and the language's stability rules have
  to work for all of them. npm in particular treats minor/patch breakage as an
  upstream bug, so wildcard-driven conflicts on additive package updates would
  be read there as buggy packages, not as users accepting a WESL-specific
  tradeoff. The defaults can't be split per-ecosystem; even libraries that
  aren't actively cross-published inherit the same rules.
- **Mixed-language ownership.** In host applications, dependency updates are
  often routine maintenance handled by someone other than the shader author. A
  wildcard conflict can land on a teammate who did not cause it and may not be
  best positioned to fix shader-side breakage.
- **Shader test coverage.** Shader test coverage is often thinner than
  application-code coverage, and some failures are visual or runtime-dependent.
  Fewer tests and WGSL's comparatively small type system mean that wildcard
  conflicts are less likely to be caught at the moment a dependency is updated.
- **Single namespace.** WGSL has a single namespace for types and values, and no
  namespace construct or object-style surface to limit the scope of wildcarded
  names after import. There are fewer places for names to coexist harmlessly.

These concerns motivate guardrails for WESL wildcard defaults.

## Wildcards in WESL: when to allow, when to gate

WESL keeps wildcards available because some libraries are designed to feel
pervasive. Game engines, test frameworks, and math libraries expect a
domain-specific API where prefixing every call with `test::expect::` or similar
would obscure the shader rather than help it. Concise import syntax matters even
where an IDE can autocomplete: not every editor has a language server, and long
import blocks add noise regardless of how they were typed.

But the concerns in
[Aren't wildcards an anti-pattern?](#arent-wildcards-an-anti-pattern) still
apply, especially across package boundaries. WESL's defaults try to keep the
benefits while limiting the risk:

- **Not every public module suits wildcards.** Modules with a fast-growing API
  or with generic names (`Buffer`, `Result`) are fine to import by name but
  hazardous to wildcard.
- **Authors can signal which modules are curated for wildcards.** An explicit
  `@!wildcardable` marker lets library authors tell consumers (and tools) which
  modules they've designed for wildcard use. It also gives tooling a hook for
  lints around generic names, builtin shadowing, churn-prone additions, etc.
- **Defaults shape the ecosystem.** Red/yellow squiggles and linter messages
  teach safe wildcard practice to new and part-time shader authors more reliably
  than community blogs or documentation.
- **Advanced users are not blocked.** Within a package, wildcard imports are
  unrestricted; externally, wildcard-importing a non-`@!wildcardable` module is
  possible via
  [standard diagnostic controls](Imports.md#suppressible-diagnostics). The
  default tunes the path of least resistance, but doesn't block users who
  intentionally accept the risk.
