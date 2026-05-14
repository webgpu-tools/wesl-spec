# Visibility Design
This document records design decisions behind WESL's visibility system. The
normative spec lives in [Visibility.md](Visibility.md).

## Contents

* [Why three levels and not two](#why-three-levels-and-not-two)
* [Why three levels and not four or more](#why-three-levels-and-not-four-or-more)
* [Why package by default?](#why-package-by-default)
* [Why `@public` and `@private`?](#why-public-and-private)
* [Why re-exports cannot widen](#why-re-exports-cannot-widen)
* [Future possibilities](#future-possibilities)

## Why three levels and not two

A two-level system (public / private) is feasible: Zig gets by with just `pub`
and an unannotated default, and small projects rarely miss more. Three levels
pay off because the package boundary and the module boundary are genuinely
separate questions every author asks ("what's part of my external API" vs.
"what's shared within the package"), and with only two levels they collapse into
one.

The result is cheaper internal refactors and a smaller external API. With a
middle level, a cross-file helper stays inside the package without inflating
what consumers depend on, and moving it between internal files is not a breaking
change to anyone outside. With only public/private, that helper is either
exposed (its path becomes part of the contract) or file-local (no cross-file
sharing at all).

WGSL compatibility nudges in the same direction. WGSL has no visibility
keywords, so an unannotated `.wgsl` file participates with whatever the default
is, and a middle level is the only default that gives sensible behavior here
(see [Why package by default?](#why-package-by-default)).

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

WESL stops at three because:

* **Shader packages are relatively small.** A typical WESL package is on the
  order of tens of files, not the thousands a Rust crate or C# assembly might
  contain. Finer-than-package visibility levels (Rust's `pub(super)` and
  `pub(in path)`, Swift's `fileprivate`) pay off when refactoring large
  codebases; at the WESL scale, the cost of the extra concept exceeds the
  benefit.
* **The two boundaries that matter most are the package boundary and the module
  boundary.** "What's part of my external API" and "what's an implementation
  detail of this module" are decisions every author makes. Levels in between are
  nice to have in large codebases but not essential.
* **Less for authors to learn.** Two attributes (`@public`, `@private`) plus an
  unannotated default. An author can hold the whole visibility model in their
  head without referring back to docs.

## Why package by default?

There are three plausible defaults (public, private, package), and prior art has
examples of each as the default for at least one mainstream language.

### Why not public by default?

First, public by default works against deliberate modularity, especially
important for libraries. The default would encourage authors not to bother with
a conscious choice about what's part of the contract. Accidentally exposing too
much silently leaks `clamp_to_unit` helpers, scratch constants, and internal
structs as part of the library's external API. Refactors that rename or remove
an unmarked helper become breaking changes for downstream callers.

Second, ceremony lands on the wrong case. Most items in a library aren't part of
the external API, so the author would have to write `@private` in the common
case instead of `@public` on the rare case.

### Why not private by default?

Making private the default favors strict encapsulation: every cross-file use
requires an explicit visibility marker. That discipline is attractive for
libraries with stable APIs and large codebases. It's less of a fit for typical
WESL projects, which tend to be small and internal. Four arguments push WESL
toward package as the default:

* **Encapsulation gains are smaller at WESL scale.** A typical WESL project is a
  handful of files: a vertex/fragment pair, a compute shader, perhaps a few
  helper modules. The author already knows the whole codebase, so the
  disciplined boundaries that pay off in larger code don't add as much value.

* **Don't force visibility decisions on every author.** Any non-trivial module
  shares at least some declarations with other modules in the package (helpers,
  types, constants). With private as the default, every author would have to
  mark those items explicitly, even authors who don't otherwise care about
  encapsulation. Package as the default makes encapsulation discipline available
  to projects that want it (via `@private`), without forcing typical small
  projects to do the extra work.

* **WGSL files can be imported without modification.** WGSL has no visibility
  keywords. With package as the default, a `.wgsl` file dropped into a WESL
  project has all of its top-level declarations available to the rest of the
  package, exactly as a user would expect. With private as the default, every
  WGSL file would need to be edited or re-annotated before its declarations were
  reachable, which would undermine the WESL goal of working smoothly with
  existing WGSL code.

  WESL could distinguish WESL from WGSL by file extension and give each a
  different default. WESL doesn't: it weakens the "WESL is a superset of WGSL"
  principle, and shader source loaded as a string (without a file extension)
  would still need some other way to mark the dialect.

* **Root module entry points would need explicit markers.** A root module's
  entry points, resource variables, and overrides reach the host only when
  non-private (see [Visibility.md](Visibility.md)). With private as the default,
  every one would need a visibility marker. A plain `.wgsl` file used as the
  root wouldn't work, it would expose nothing to the host. WESL could give root
  modules a different default to avoid this, but then the root behaves unlike
  every other module. With package as the default, `@fragment fn frag_main(...)`
  at root is pipeline-visible with no extra annotation, and only `@private`
  changes anything at root.

### Package by default

Package as the default keeps the external API explicit while leaving internal
sharing free. The `@public` marker stays meaningful and rare, so internal
helpers don't accidentally leak as part of the external API. Authors aren't
forced to make visibility decisions on every cross-file use, and plain WGSL
files participate without modification.

## Why `@public` and `@private`?

Two choices shape the visibility syntax: the names (`public` and `private`) and
the form (an attribute, not a bare keyword).

### The names: `public` and `private`

`public` and `private` are plain English words used as visibility markers in
most languages with a visibility system (Java, C++, C#, Kotlin, Scala, Swift).
They form a symmetric pair and read clearly to newcomers regardless of
background language.

### The form: an attribute, not a keyword

WGSL reserves `public` but not `private`. WGSL's
[reserved-words list](https://www.w3.org/TR/WGSL/#reserved-words) includes
`public`, `pub`, `priv`, `package`, `export`, and `protected`; `private` is
absent. Adopting `public` and `private` as bare visibility keywords would
require WGSL to also reserve `private`, which would break any existing shader
that uses `private` as an identifier and requires coordination with the WGSL
working group. WGSL attributes are an open namespace by design, so `@public` and
`@private` slot in as a symmetric pair without coordination.

The attribute form also matches how WESL already extends WGSL. Other
declaration-level metadata (`@wildcardable`, `@diagnostic`) is expressed as
attributes, and WGSL itself uses attributes for `@vertex`, `@workgroup_size`,
`@group`, `@binding`, and more. The cost is a small amount of visual noise (an
extra `@` on every visibility marker); shaders are already attribute-heavy
enough that this fits the local style.

WESL does not pin a position for the visibility attribute in the grammar, just
as WGSL does not pin a position for `@group` or `@workgroup_size`. By convention
(a formatting rule, not a grammar rule), the visibility attribute sits last,
immediately before the declaration keyword, so a reader scanning down the left
margin can find it without hunting through the attribute list.

### Alternatives considered

**`pub` / `priv` (Rust).** WGSL reserves both `pub` and `priv`, so they could be
adopted as bare keywords without coordination. `public` still reads more clearly
to newcomers and to authors coming from non-Rust languages, and `priv` has
little prior art outside early Rust, which dropped it.

**`export` / `hide` (TypeScript).** TypeScript uses `export` to mean "this
declaration is visible outside this module." That works in TS because TS has
only two visibility levels at the module boundary (exported / not), and because
TS modules are the privacy unit (there is no package-internal level). WESL has
three levels and needs both extremes nameable, so `export` would have to be
paired with a separate `@hide` (or similar). Keeping declaration and visibility
orthogonal (a declaration always declares; an optional attribute sets
visibility) is cleaner.

**Capitalization (Go).** Go encodes visibility in the first letter of an
identifier: capital means exported, lowercase means unexported. The rule is
elegantly minimal but locks the language to two levels (see
[Why three levels and not two](#why-three-levels-and-not-two)). It would also
impose a retroactive visibility semantics on existing WGSL identifiers, which
follow no convention tying capitalization to visibility: a `.wgsl` file dropped
into a WESL project would have its visibility silently determined by how its
names happen to be cased.

## Why re-exports cannot widen

A `@public import` cannot widen visibility: re-exporting a *package* or
`@private` item as `@public` is a hard error.

The reason is that the visibility attribute on a declaration is meant to be a
local contract. An author looking at a *package* item should be able to rely on
the attribute without scanning every `@public import` in the package for a
republication that would silently widen it. Widening would turn the attribute
into a hint rather than a guarantee: an author refactoring what they believed
was an internal helper could break downstream callers who reached it through a
widened re-export elsewhere in the package.

The cost of the rule is that items exposed through a curated prelude must be
declared `@public` at their definition, leaving the original module path
reachable too. Restricting reach to a single canonical path is sketched under
[Future possibilities](#future-possibilities).

## Future possibilities

### `@package import`

Only `@public import` is specified for now. A `@package import` form could cover
a "re-export within the package, hide externally" use case.

### `@package` attribute on declarations

A `@package` attribute would make the middle level explicit on a declaration,
rather than leaving it implied by the absence of `@public` or `@private`. The
unannotated form covers every present use, so the attribute isn't part of the
current design; it would mainly help where an author or a tool wants to see that
a `package` level was chosen deliberately rather than left unmarked.

### Submodule re-export

`@public import` re-exports individual items, not whole submodules. Module
re-export would let a package present an internal `mylib::internal::math` module
under a cleaner path like `mylib::math`.

### Canonical prelude path

The curated-prelude pattern leaves an item's original module path reachable
alongside the prelude path, so internal module layout is part of what consumers
can reach. A `@canonical` annotation on the re-export could mark the prelude
path as the intended one, letting tooling steer consumers to it (and flag use of
the original path) without any change to name resolution.

### re-export widening from root

A plain `.wgsl` file with an entry point cannot be aggregated into a root module
via `@public import` without modification (see
[Aggregating entry points](Visibility.md#aggregating-entry-points)). A
relaxation would let the root module `@public import` *package* items from the
same package, since the root defines the package's external and pipeline-visible
API. The cost is loss of orthogonality: what `@public import` accepts would
depend on whether the importer is the root.

### Wildcard re-export

`@public import path::*` would re-export every public item of a target module.
It is deferred because resolution would have to walk a set of modules
recursively (modules can form cycles, so resolution would need to track visited
modules), and it interacts awkwardly with potential future parameterized modules
design, while enabling nothing that named re-exports cannot already express. Two
questions need answers before it could be specified: how latent collisions
across a library's wildcard re-exports are caught at publish time rather than by
consumers, and what happens when wildcard re-exporting from a module that itself
has wildcard imports. Publishing tools (see
[#183](https://github.com/webgpu-tools/wesl-spec/issues/183)) could also expand
wildcard re-exports at publish time into explicit named re-exports.
