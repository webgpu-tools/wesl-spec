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
* [Future possibilities](#future-possibilities)
  * [`package import`](#package-import)
  * [`package` keyword on declarations](#package-keyword-on-declarations)
  * [Submodule re-export](#submodule-re-export)
  * [Canonical prelude path](#canonical-prelude-path)
  * [Re-export widening from root](#re-export-widening-from-root)
  * [Wildcard re-export](#wildcard-re-export)
  * [Diagnosing leaked types](#diagnosing-leaked-types)

## Why three levels and not two

A two-level system (public / private) is feasible: Zig gets by with just `pub`
and an unannotated default, and small projects rarely miss more. Three levels
help because the package boundary and the module boundary are genuinely separate
questions every author may ask ("what's part of my external API" vs. "what's
shared within the package"), and with only two levels they collapse into one.

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
* **Less for authors to learn.** Two keywords (`public`, `private`) plus an
  unannotated default. An author can hold the whole visibility model in their
  head without referring back to docs.

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

* **Root module entry points would need explicit markers.** A root module's
  entry points, resource variables, and overrides reach the host only when
  non-private (see [Visibility.md](Visibility.md)). With private as the default,
  every one would need a visibility marker. With package as the default,
  `@fragment fn frag_main(...)` at root is pipeline-visible with no extra
  keywords or annotations.

  WGSL's stability guarantees add a further constraint, because WESL features
  are designed to be adoptable into WGSL. The current design uses the same rules
  and keywords for pipeline visibility as for visibility between modules. Under
  private by default, unmarked entry points and unmarked overrides would be
  invisible to the host, so existing WGSL programs would break on the day a WGSL
  implementation rolled out visibility.

  Avoiding that breakage would mean designing pipeline visibility to work
  differently from module visibility, perhaps with separate keywords, or with
  different meanings for the same keywords in the root module versus other
  modules. That is another layer of complexity that default package visibility
  avoids entirely.

* **WGSL compatibility would be awkward.** WGSL has no visibility keywords. With
  package as the default, a `.wgsl` file dropped into a WESL project has all of
  its top-level declarations available to the rest of the package, exactly as a
  user would expect. With private as the default, every WGSL file would need to
  be edited or re-annotated before its declarations were reachable, which would
  undermine the WESL goal of working smoothly with existing WGSL code.

  WESL could distinguish WESL from WGSL by file extension and give each a
  different default. But that would weaken the "WESL is a superset of WGSL"
  principle, and shader source loaded as a string (without a file extension)
  would still need some other way to mark the dialect. If WGSL adopts
  visibility, the defaults would effectively flip for users. We'd need some
  alternative to avoid that strangeness.

* **Don't force visibility decisions on every author.** Any non-trivial module
  shares at least some declarations with other modules in the package (helpers,
  types, constants). With private as the default, every author would have to
  mark those items explicitly, even authors who don't otherwise care about
  encapsulation. Package as the default keeps that cost off the common case
  while leaving `private` available to authors who want stricter boundaries.

  Major languages disagree on whether this cost is worth paying. Some (Rust,
  TypeScript) require markers for cross-file sharing within a package; others
  (Java, Scala, Kotlin, Swift) do not. There is no settled best answer, so WESL
  weighs the choice for its own users.

* **WESL projects are small, even when frameworks are big.** Shader projects are
  generally much smaller than projects in general purpose programming languages.
  A typical WESL author writes a vertex/fragment pair, a compute shader, and
  maybe a few helpers of their own, even when building using a large framework.

  Authors of smaller applications can be a little lighter weight and a little
  more concise by mostly not worrying about visibility. Just using default
  package visibility with no markers is probably enough for many smaller apps.
  The three available visibility levels add controls that larger projects will
  welcome, and small projects can mostly skim past the complexity.

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

WGSL reserves `public` as a reserved word, and `private` is a context-sensitive
keyword in WGSL. Adopting `public` and `private` as bare visibility keywords
therefore needs no special accommodation: `public` is already off-limits as an
identifier, and `private` remains permitted as an identifier in non-visibility
positions.

A visibility keyword may prefix any item declaration. By convention, it sits
between any attributes and the leading declaration keyword, so a reader scanning
down the left margin can find it without hunting through the attribute list.

### Why not `pub`?

`public` is a plain English word that reads clearly to newcomers; `pub` is a
Rust-specific abbreviation. Most languages with visibility keywords use
`public`. It also pairs grammatically with `private`, which `pub` does not. The
cost (a few extra characters per declaration) is negligible.

### Why not `export`?

TypeScript uses `export` to mean "this declaration is visible outside this
module." That works in TS because TS has only two visibility levels at the
module boundary (exported / not), and because TS modules are the privacy unit
(there is no package-internal level). WESL has three levels and needs both
extremes nameable, so `export` would have to be paired with a separate `hide`
keyword (or similar). Keeping declaration and visibility orthogonal (a
declaration always declares; an optional keyword sets visibility) is cleaner.

### Why not capitalization syntax?

Go encodes visibility in the first letter of an identifier: capital means
exported, lowercase means unexported. The rule is elegantly minimal but locks
the language to two levels (see
[Why three levels and not two](#why-three-levels-and-not-two)). It would also
impose a retroactive visibility semantics on existing WGSL identifiers, which
follow no convention tying capitalization to visibility: a `.wgsl` file dropped
into a WESL project would have its visibility silently determined by how its
names happen to be cased.

### Why not `@public` and `@private`?

Bare keywords read more clearly than `@public`/`@private` attributes and avoid
the persistent `@` noise on every visibility marker, and they require no special
accommodation given WGSL's existing reserved-word and context-sensitive-keyword
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

## Future possibilities

### `package import`

Only `public import` is specified for now. A `package import` form could cover a
"re-export within the package, hide externally" use case.

### `package` keyword on declarations

A `package` keyword would make the middle level explicit on a declaration,
rather than leaving it implied by the absence of `public` or `private`. The
unannotated form covers every present use, so the keyword isn't part of the
current design; it would mainly help where an author or a tool wants to see that
a `package` level was chosen deliberately rather than left unmarked.

### Submodule re-export

`public import` re-exports individual items, not whole submodules. Module
re-export would let a package present an internal `mylib::internal::math` module
under a cleaner path like `mylib::math`.

### Canonical prelude path

The curated-prelude pattern leaves an item's original module path reachable
alongside the prelude path, so internal module layout is part of what consumers
can reach. A `@canonical` annotation on the re-export could mark the prelude
path as the intended one, letting tooling steer consumers to it (and flag use of
the original path) without any change to name resolution.

### Re-export widening from root

A plain `.wgsl` file with an entry point cannot be aggregated into a root module
via `public import` without modification (see
[Aggregating entry points](Visibility.md#aggregating-entry-points)). A
relaxation would let the root module `public import` *package* items from the
same package, since the root defines the package's external and pipeline-visible
API. The cost is loss of orthogonality: what `public import` accepts would
depend on whether the importer is the root.

### Wildcard re-export

`public import path::*` would re-export every public item of a target module. It
is deferred because resolution would have to walk a set of modules recursively
(modules can form cycles, so resolution would need to track visited modules),
and it interacts awkwardly with potential future parameterized modules design,
while enabling nothing that named re-exports cannot already express. Two
questions need answers before it could be specified: how latent collisions
across a library's wildcard re-exports are caught at publish time rather than by
consumers, and what happens when wildcard re-exporting from a module that itself
has wildcard imports. Publishing tools (see
[#183](https://github.com/webgpu-tools/wesl-spec/issues/183)) could also expand
wildcard re-exports at publish time into explicit named re-exports.

### Diagnosing leaked types

A `public` declaration may name a less-visible type in its signature or fields
(see
[Referring to less-visible declarations](Visibility.md#referring-to-less-visible-declarations)).
This is usually intentional, but it can also be an accident that quietly narrows
what a consumer can do with a public API. Tools could warn when a `public`
declaration mentions a less-visible type, with a suppression such as
`@diagnostic(off, leaked_type)` for the deliberate cases.

The current spec permits it with no diagnostic. The leak case is rare and its
shape in real WESL code is unproven, so prescribing a warning now would commit
to the concept before practice shows whether it is wanted, and which uses are
deliberate. A warning is purely additive, so it can be introduced later once
usage is clearer without breaking any program that links today. (Rust began with
a strict rule here and relaxed it in
[RFC 2145](https://rust-lang.github.io/rfcs/2145-type-privacy.html); Go and
TypeScript permit the leak and lean on lints.)
