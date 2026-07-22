# Visibility Design

This document records design decisions behind WESL's visibility system. The
normative spec lives in [Visibility.md](Visibility.md).

## Contents

* [Why three levels and not two](#why-three-levels-and-not-two)
* [Why three levels and not four or more](#why-three-levels-and-not-four-or-more)
* [Why package by default?](#why-package-by-default)
  * [Why not public by default?](#why-not-public-by-default)
  * [Why not private by default?](#why-not-private-by-default)
  * [Package by default](#package-by-default)
* [Why `public` and `private` syntax?](#why-public-and-private-syntax)
  * [Why not `pub`?](#why-not-pub)
  * [Why not `export`?](#why-not-export)
  * [Why not capitalization syntax?](#why-not-capitalization-syntax)
  * [Why not `@public` and `@private`?](#why-not-public-and-private)
* [Why re-exports cannot widen](#why-re-exports-cannot-widen)
* [Why not infer the pipeline-visible API?](#why-not-infer-the-pipeline-visible-api)
* [Future possibilities](#future-possibilities)
  * [`package import`](#package-import)
  * [`package` keyword on declarations](#package-keyword-on-declarations)
  * [Module re-export](#module-re-export)
  * [Canonical prelude path](#canonical-prelude-path)
  * [Re-export widening from root](#re-export-widening-from-root)
  * [Wildcard re-export](#wildcard-re-export)
  * [Member visibility](#member-visibility)
  * [Diagnosing less-visible types in public
    APIs](#diagnosing-less-visible-types-in-public-apis)

## Why three levels and not two

A two-level system (public / private) is feasible: Zig uses `pub` and an
unannotated default. Three levels help because the package and module boundaries
represent separate decisions: what belongs to the external API, and what can be
shared within the package. With only two levels, those decisions collapse into
one.

The result is cheaper internal refactors and a smaller external API. With a
middle level, a cross-file helper stays inside the package without inflating
what consumers depend on, and moving it between internal files is not a breaking
change to anyone outside. With only public/private, that helper is either
exposed (its path becomes part of the contract) or file-local (no cross-file
sharing at all).

WGSL compatibility also makes a middle level useful. Unannotated WGSL
declarations need to remain available across files without automatically
becoming part of a library's external API (see
[Why package by default?](#why-package-by-default)).

The keyword set has plenty of prior art: roughly Rust's `pub(crate)`, Java's
package-private (default), Scala's `private[pkg]`, Swift's `package` (5.9).
Kotlin, Slang, and Swift also offer `internal` at their compilation-unit
granularity (Kotlin module, Swift module, Slang module). Languages that stop at
two keyword levels often recover the third structurally: Go's `internal/`
directories restrict a subtree to its enclosing package, and TypeScript packages
routinely keep a curated public entry point (an `exports` map plus a barrel
`index.ts`) distinct from the modules they `export` for cross-file use but never
re-export.

## Why three levels and not four or more

WESL stops at three because module and package are the two structural
encapsulation boundaries already present in the language. Finer scopes such as
Rust's `pub(super)` and `pub(in path)` would make access depend on the module
hierarchy and require additional syntax for naming that scope. That extra
coupling is not needed to express the boundaries this proposal targets. Two
keywords (`public`, `private`) plus an unannotated default cover them.

## Why package by default?

There are three plausible defaults (public, private, package), and prior art has
examples of each as the default for at least one mainstream language.

### Why not public by default?

First, the external API grows by accident. With public as the default, every
unmarked `clamp_to_unit` helper, scratch constant, and internal struct silently
becomes part of a library's external API. Refactors that rename or remove an
unmarked helper then become breaking changes for downstream callers, even though
the author never intended the helper to be part of the contract. The default
works against deliberate modularity by encouraging authors not to make a
conscious choice about what's public.

Second, ceremony lands on the wrong case. Most items in a library aren't part of
the external API, so the author would have to write `private` in the common case
instead of `public` on the rare case.

### Why not private by default?

Making private the default favors strict encapsulation: every cross-file use
requires an explicit visibility marker. Encapsulation discipline by default is
handy for larger teams working on larger codebases. But private by default
wouldn't work as well for WESL for a few reasons:

* **Module entry points would need explicit markers.** The entry module's
  entry points, resource variables, and overrides reach the host only when
  non-private (see [Visibility.md](Visibility.md)). With private as the default,
  every one would need a visibility marker. With package as the default,
  `@fragment fn frag_main(...)` at root is pipeline-visible with no extra
  keywords or annotations.

  WESL features are also designed for possible adoption into WGSL. A private
  default would make existing, unmarked WGSL entry points and overrides
  invisible to the host. Avoiding that incompatibility would require separate
  defaults or visibility rules for the entry module. Package visibility avoids that
  special case.

* **WGSL compatibility would be awkward.** WGSL has no visibility keywords. With
  package as the default, a `.wgsl` file dropped into a WESL project has all of
  its top-level declarations available to the rest of the package, exactly as a
  user would expect. With private as the default, every WGSL file would need to
  be edited or re-annotated before its declarations were reachable, which would
  undermine the WESL goal of working smoothly with existing WGSL code.

  WESL could distinguish WESL from WGSL by file extension and give each a
  different default. That would weaken the "WESL is a superset of WGSL"
  principle, and shader source loaded as a string has no file extension to
  distinguish its dialect.

* **Don't force visibility decisions on every author.** Many modules share
  helpers, types, or constants with other modules in the package. With private
  as the default, authors would have to mark each of those items explicitly.
  Package as the default keeps that cost off the common case while leaving
  `private` available for stricter boundaries.

  Major languages disagree on whether this cost is worth paying. Some (Rust,
  TypeScript) require markers for cross-file sharing within a package; others
  (Java, Scala, Kotlin, Swift) do not. There is no settled best answer, so WESL
  weighs the choice for its own users.

### Package by default

Package as the default keeps the external API explicit while leaving internal
sharing free. The `public` marker stays meaningful and rare, so internal helpers
don't accidentally leak as part of the external API. Authors aren't forced to
make visibility decisions on every cross-file use, plain WGSL files participate
without modification, and the design is adoptable by a future version of WGSL.

## Why `public` and `private` syntax?

`public` and `private` are plain English words used as visibility markers in
most languages with a visibility system (Java, C++, C#, Kotlin, Scala, Swift).
They form a symmetric pair and read clearly to newcomers regardless of
background language.

WGSL reserves `public` as a reserved word, and `private` is a context-dependent
name in WGSL. Adopting `public` and `private` as bare visibility keywords
therefore needs no special accommodation: `public` is already off-limits as an
identifier, and `private` remains permitted as an identifier in non-visibility
positions.

A visibility keyword may prefix any item declaration. The grammar places it
between any attributes and the leading declaration keyword, so a reader always
finds it next to the declaration it modifies rather than hunting through the
attribute list.

### Why not `pub`?

`public` is a plain English word that reads clearly to newcomers; `pub` is an
abbreviation used by Rust and Zig. Most languages with visibility keywords use
`public`. It also pairs grammatically with `private`, which `pub` does not. The
cost (a few extra characters per declaration) is negligible.

### Why not `export`?

TypeScript uses `export` to mean "this declaration is visible outside this
module." In WESL, `export` would name the public extreme while `private` named
the other, producing an asymmetric pair around the package default. `public` and
`private` state the two visibility levels directly, and `public import` makes
re-exporting an imported item explicit.

### Why not capitalization syntax?

Go encodes visibility in the first letter of an identifier: capital means
exported, lowercase means unexported. That gives only a binary distinction, so
WESL would still need separate syntax for its third level. It would also impose
visibility semantics on existing WGSL identifiers, which follow no convention
tying capitalization to visibility: a `.wgsl` file used in a WESL project would
have its visibility silently determined by how its names happen to be cased.

### Why not `@public` and `@private`?

Bare keywords read more clearly than `@public`/`@private` attributes and avoid
the persistent `@` noise on every visibility marker. They require no special
accommodation given WGSL's existing reserved-word and context-dependent-name
status for the two names.

## Why re-exports cannot widen

A `public import` cannot widen visibility: re-exporting a *package* or `private`
item as `public` is a hard error.

The reason is that the visibility keyword on a declaration is meant to be a
local contract. An author looking at a *package* item should be able to rely on
the keyword without scanning every `public import` in the package for a
republication that would silently widen it. Widening would turn the keyword into
a hint rather than a guarantee: an author refactoring what they believed was an
internal helper could break downstream callers who reached it through a widened
re-export elsewhere in the package.

The cost of the rule is that items exposed through a curated prelude must be
declared `public` at their definition, leaving the original module path
reachable too. Restricting reach to a single canonical path is sketched under
[Future possibilities](#future-possibilities).

The rule also applies to an alias whose target resolves through aliases to a
declared structure type. A WGSL alias introduces another name for the same type
and preserves its value constructors and members. When `InternalStuff`
ultimately denotes a less-visible structure type, `public alias Exposed =
InternalStuff;` would make that type nameable and constructible at `public`
visibility even though its declaration restricts it. Intermediate aliases do
not change the type or impose their own visibility on a later alias, so a
`public` alias may name a less-visible alias for a `public` structure type.

A less-visible type that appears in a `public` function signature, a member of a
`public` struct, or within an alias target whose resolved target is composite is
different: none of those declarations introduces another name for the
less-visible type itself. Consumers can use values they are handed but cannot
name that type or construct new values of it. Those uses remain permitted; see
[Diagnosing less-visible types in public
APIs](#diagnosing-less-visible-types-in-public-apis).

## Why not infer the pipeline-visible API?

Pipeline visibility could be inferred from the linked program, exposing every
entry point and every resource variable or `override` they statically access.
That would weaken library boundaries. A library should be able to hide the
libraries it depends on behind its own API. With inference, pipeline-relevant
items from those dependencies would leak into the application's host interface,
including items enabled by configurations that the library neither controls nor
tests.

Inferred entry points and overrides identified by name would also require
host-visible names. Their WESL declaration paths use `::`, which WGSL
identifiers cannot contain, while their bare names can collide in WGSL's flat
namespace. A linker must therefore encode the paths or otherwise mangle the
names. Even if WESL standardized that encoding, host code could not use WESL
paths directly. It would have to reproduce the encoding or use a WESL-aware host
library to translate them, and moving a declaration could still change its
host-facing name. Because host references may be strings in any host language,
WESL tools could not reliably find or update them when that name changes.

Instead, WESL keeps each boundary explicit. A library can use a `public` item
from a dependency with a bare `import` without passing it on, or re-export it
with a `public import`. The entry module makes a separate choice for the host: it
exposes its non-private pipeline-relevant items directly and selected library
items through `public import`. This lets each application and library decide
which dependency items cross its own API boundary.

This design requires the entry module to name exposed items explicitly. A small
program may declare them in the entry module; a larger program can expose items
from other modules with `public import`.

## Future possibilities

### `package import`

Re-exports currently always have `public` visibility. A `package import` form
could cover a "re-export within the package, hide externally" use case.

### `package` keyword on declarations

A `package` keyword would make the middle level explicit on a declaration,
rather than leaving it implied by the absence of `public` or `private`. The
unannotated form covers every present use, so the keyword isn't part of the
current design; it would mainly help where an author or a tool wants to see that
a `package` level was chosen deliberately rather than left unmarked.

### Module re-export

`public import` re-exports individual items, not whole modules. Module re-export
would let a package present `mylib::internal::math` under a shorter path such as
`mylib::math`.

### Canonical prelude path

The curated-prelude pattern leaves an item's original module path reachable
alongside the prelude path, so internal module layout is part of what consumers
can reach. A `@canonical` annotation on the re-export could mark the prelude
path as the intended one, letting tooling steer consumers to it (and flag use of
the original path) without any change to name resolution. Stronger designs that
make the original path unreachable (module visibility controls) are collected in
[#202](https://github.com/webgpu-tools/wesl-spec/issues/202).

### Re-export widening from root

A plain `.wgsl` file with an entry point cannot be aggregated into an entry module
via `public import` without modification (see
[Aggregating entry points](Visibility.md#aggregating-entry-points)). A
relaxation would let the entry module `public import` *package* items from the
same package, since the root already chooses the pipeline-visible API. The cost
is loss of orthogonality: what `public import` accepts would depend on whether
the importer is the root. Root status belongs to a link invocation, not to the
module itself; the same module can be a root in one link and an ordinary module
in another. Source-only tools such as the language server could not check that
rule without knowing how the module will be linked.

### Wildcard re-export

`public import path::*` would re-export every public item of a target module. It
is deferred because resolution would have to walk a set of modules recursively
(modules can form cycles, so resolution would need to track visited modules),
and it interacts awkwardly with a potential future parameterized-module design,
while enabling nothing that named re-exports cannot already express. Two
questions need answers before it could be specified: how latent collisions
across a library's wildcard re-exports are caught at publish time rather than by
consumers, and what happens when wildcard re-exporting from a module that itself
has wildcard imports. Publishing tools (see
[#183](https://github.com/webgpu-tools/wesl-spec/issues/183)) could also expand
wildcard re-exports at publish time into explicit named re-exports.

### Member visibility

The current design does not restrict struct member access. A module that obtains
a struct value may read its members even if the struct type is not visible in
that module.

A future extension could support opaque or partially opaque structs by
restricting member access and value construction separately from type
visibility. Unmarked members would remain accessible for WGSL compatibility. The
syntax and whether restrictions use module or package boundaries remain open.

### Diagnosing less-visible types in public APIs

A less-visible type may appear in a `public` function signature, a member of a
`public` struct, or within a `public` alias target whose resolved target is a
composite type (see
[Referring to less-visible declarations](Visibility.md#referring-to-less-visible-declarations)).
When a public API exposes a less-visible struct value, its readable members can
become source-compatibility constraints for the author even though the struct
declaration is not public. A future diagnostic could warn when a `public`
declaration mentions a less-visible type, with a suppression such as
`@diagnostic(off, leaked_type)` for the deliberate cases.
