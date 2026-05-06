# Summary
We propose adding an importing mechanism to the WGSL shading language as an extension.

# Motivation
When writing bigger WGSL shaders, one tends to **split the code into reusable pieces**. Examples are re-using a set of lighting calculation functions and sharing a struct between two compute shaders.

However, current WGSL tooling is not built for this. The official language purposefully does not include this feature, nor does it provide adjacent features like namespaces, and there is no standardized extension yet.

We also should account for **importing shader from libraries**. Ideally, users could upload WGSL shaders to existing package managers, which other users could then consume.

Finally, we want **multiple tools** which can compile WGSL-with-imports down to raw WGSL. Using WGSL-with-imports both in Rust projects and in web projects should be possible.

# Guide-level explanation
The `import` statement extension brings items or entire modules into scope. Import statements map to files with minimal searching.

```wgsl
// Importing a single function using a relative path
import super::lighting::pbr;

// Importing multiple items
import my::geom::sphere::{ draw, default_radius as foobar };

// Imports a whole module. Use it with `bevy_ui::name`
import bevy_ui;

// Import all items from another module
import bevy::prelude::*;
import wgsl_test::prelude::*;
```

These can then be used anywhere in the source code.

```wgsl
fn main() {
    bevy_ui::quad(vec2f(0.0, 1.0), 1.0);
    let a = draw(3);
}
```

Both `bevy_ui` and `my` are packages in the current project. Language servers and related tools can look in a `wesl.toml` file to find the location of the packages. This lets libraries be published to package managers, and users can import them with a simple syntax.

Recursive import definitions are also supported, which leads to shorter import statements.

```wgsl
import bevy_pbr::{
  forward_io::VertexOutput,
  pbr_types::{PbrInput, pbr_input_new},
  pbr_bindings
};
```

To find the relevant items, the following algorithm is used:

Proceeding left to right through the path segments, consider the segments `prev` and `seg`.
1. if `prev.wesl` exists and includes WESL elements, check if `seg` is one of those elements, e.g. `fn seg` or `namespace seg`.
2. else if the directory `prev/` exists, check to see if the file `seg.wesl` or the directory `seg/` is in the `prev/` directory;
3. error if a `seg` is not found.

# Reference-level explanation
A WESL program is composed of a tree of WESL modules.

Imports must appear as the first items in a WESL file. They can import entire modules or individual "importable items" (see [GLOSSARY](GLOSSARY.md)).

### Grammar

An import statement is parsed with the following grammar, with spaces and comments allowed between tokens:

```ebnf
translation_unit:
| import_statement* global_directive* global_decl* 

import_statement:  
| 'import' import_relative? (import_collection | import_path_or_item) ';'  

import_relative:
| 'package' '::' | 'super' '::' ('super' '::')*

import_path_or_item:
| ident '::' (import_collection | import_path_or_item) 
| ident ('as' ident)?
| '*'

import_collection:
| '{' (import_path_or_item) (',' (import_path_or_item))* ','? '}'
```

Where `translation_unit` and `ident` are defined in the WGSL grammar.
`ident`s must not be current WGSL keywords. `ident`s also must not be 
current WESL keywords: `as`, `import`, `package`, `super`, or `self`. 
Reserved words that are
not current keywords are allowed, 
but not recommended.
Lint tools may optionally warn when reserved words are used. 

An item import imports a single item. The item can be renamed with the `as` keyword.

An import collection imports multiple items, and allows for nested imports.

A wildcard import imports all top-level declarations from a module. Submodule names and submodule contents are not imported.

WESL also extends WGSL's `global_directive` rule with a `module_attribute` form (used by `@wildcardable`; see [Wildcard imports](#wildcard-imports)):

```ebnf
global_directive:
| ... // existing WGSL forms
| module_attribute

module_attribute:
| '@' ident ';'
```

The `@` prefix avoids conflict with WGSL identifiers.

### Import resolution algorithm

To resolve the import, the recursive structure is flattened out. This means turning every `import_collection` into multiple separate imports, ending with the items.
For instance, `import a::{b, c::{d, e as f}};` would be turned into

```wesl
import a::b;
import a::c::d;
import a::c::e as f;
```

Then, one iterates over each segment from left to right, and looks it up one by one.

1. We start with the first segment.
    * `super` refers to the parent module. Can be repeated to go up multiple parent modules. Exiting the root is an error.
    * `package` refers to the top level module of the current package.
    * `ident` must be a known package, usually found in the `wesl.toml` file. It refers to the top level module of that package.
2. We take that as the "current module".
3. We repeatedly look at the next segment.
    1. Item in current module: Take that item. We must be at the last segment, otherwise it's an error.
    2. Wildcard import: Take all items in the current module.
    3. (Else if re-exported or inline module in current module: We continue with that module.)
    4. Else go to `current module path/ident.wesl`
       * File found: We take that file as the current module.
       * File not found: We assume an empty module as the current module, and continue with that.
       * (Re-exporting changes the path.)
       * (Inline modules do not have a path.)

To get an absolute path to a module, one follows the algorithm above. In step 1, one takes the known absolute path of the `super` module, or the package.
The absolute path of the `super` module is always known, since the first loaded WESL file must always be the root module, and children are only discovered from there.

Once the import has been resolved, the last segment, or its `as` alias, is brought into scope.


### Example

For example

```wgsl
import bevy_pbr::forward_io::VertexOutput;
```

This first looks for `bevy_pbr.wesl`.
`bevy_pbr.wesl` is found, and doesn't contain an item named `forward_io`.
Thus, we go to `bevy_pbr/forward_io.wesl`. It contains a struct named `VertexOutput`.

Another example

```wgsl
import super::shadowmapping;
```

Assume that the current module lives at `shaders/lighting.wesl`. We first go to the super module at `shaders.wesl`. We then look for an item called `shadowmapping` in `shaders.wesl`.
After not finding it, we look for a module `shadowmapping` at `shaders/shadowmapping.wesl`.

## `wesl.toml`
The [`wesl.toml`](WeslToml.md) file provides linker configuration options affecting the import resolution algorithm. It can customize:

* The root directory,
* Available package dependencies,
* A file whitelist and/or blacklist.

## Filesystem Resolution
To resolve a module on a filesystem, one follows the algorithm above.
The root folder, or the root module, needs to be provided to the linker. This is currently a linker-specific API, and may change once we introduce a `wesl.toml`.

Linkers should fall back to `.wgsl` files when a `.wesl` file cannot be found.

Due to filesystem limitations, it can happen that WESL idents are invalid file or folder names.
Notable examples are `CON, PRN, AUX, NUL, COM1 - COM9, LPT1 - LPT9` on Windows, and Windows being case-insensitive.
We do not take these restrictions into account, instead we just recommend that WESL programmers avoid these special names.

## Inline Usage
The syntax can also be used inline. To do so, we extend
the [WGSL grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent) as follows.

We introduce

```ebnf
full_ident:
| import_relative? ident ('::' ident)*
```

and then replace `ident` with `full_ident` in the following places:

```ebnf
core_lhs_expression:
| full_ident
| ...

for_init: 
| full_ident func_call_statement.post.ident
| ...

for_update:
| full_ident func_call_statement.post.ident
| ...

global_decl: 
| 'alias' ident '=' full_ident ...
| ...

primary_expression: 
| full_ident template_elaborated_ident.post.ident
| full_ident template_elaborated_ident.post.ident argument_expression_list
| ...

statement: 
| ...
| full_ident template_elaborated_ident.post.ident argument_expression_list ';'
| ...

type_specifier:
| full_ident ...
```

When resolving inline imports, we can also use modules that were imported. This means that the import resolution algorithm is extended to
1. We start with the first segment.
    * ...
    * `ident` refers to an identifier that is in scope. If it is a module, we start with that. If it is not a module, it is an error. If there is none, we fall back to the packages.

Examples

```wesl
import foo::bar;
fn main() {
    let a = bar::baz; // Uses bar from the import above
    let b = bevy::main(); // Uses the known bevy package
}
```

## Cyclic Imports
Cyclic imports are allowed.

However, the following is still illegal.

```wesl
// foo.wesl
import bar::b;
const a = b + 1;
```

```wesl
// bar.wesl
import foo::a;
const b = a + 1;
```

(a depends on b which depends on a)

Basic linker implementations do not need to check for this. Generating broken code and letting the underlying shader compiler throw an error is fine.

## Wildcard imports

Wildcard imports bring all items from another module into the importing module's
scope.

Users can wildcard import:
- from any other module in the local package.
- from any external library module where the library author has added a
  `@wildcardable` directive.

Wildcard importing brings some stability risk when the imported module adds to
its API. The newly introduced names may conflict with other names in the
importer's namespace (from local definitions and other imports), leading to
compiler warnings or errors. To help mitigate this risk, WESL provides a
`@wildcardable` annotation that library authors can place on modules that are
designed for wildcard importing.

Advanced users who wish to wildcard import from external modules not marked as
`@wildcardable` can do so by annotating the import statement with
`@diagnostic(off, wildcard_import)`, accepting some additional upgrade risk.

```wesl
// wildcard import from a @wildcardable external
import bevy::prelude::*;
import wgsl_test::prelude::*;

// wildcard import from within the local package
import package::utils::*;
import super::fun::*;
```

### `@wildcardable` directive

Library authors mark modules they intend for library consumers to wildcard
import with `@wildcardable`:

```wesl
// math.wesl  (in a library)
@wildcardable;

fn dot2(a: vec2f, b: vec2f) -> f32 { return a.x*b.x + a.y*b.y; }
fn cross2(a: vec2f, b: vec2f) -> f32 { return a.x*b.y - a.y*b.x; }
```

`@wildcardable` is a module attribute (see [Grammar](#grammar)). It must
appear after any imports and before any global declarations, and applies only
to the module it appears in.

### Recommendations for `@wildcardable` modules

Because every name in a `@wildcardable` module is a potential collision in
importer code, library authors should curate these modules carefully.

**Add hesitantly.** Additions to a `@wildcardable` module are semver minor
version bumps but can break users who have local declarations or import other
`@wildcardable` modules.
- **Bundle** additions into a major release when one is upcoming.
- **Document** additions clearly in changelogs so downstream users debugging
  unexpected name resolution can trace them.

**Compose with re-exports.** See [Re-exports](#re-exports) (TBD) to collect
items from other modules into a single `@wildcardable` module for user
convenience.

**Avoid generic names.** Prefer domain-specific names. Common names like
`Buffer`, `Config`, `Result`, `Vec`, etc. are more likely to collide with user
applications.

**Don't shadow WGSL builtins.** Names like `vec3`, `clamp`, `inverseSqrt` have
expected semantics that oughtn't be implicitly overridden with wildcards.
Similarly, avoid experimental Naga/Dawn/Safari builtins.
- If a future WGSL update adds a conflicting builtin name, plan to update the
  `@wildcardable` module to rename the conflicting item.

### Library-to-library wildcard imports

Libraries that wildcard import from other libraries raise special concerns. If a
user's package manager chooses a newer version of the imported-from library,
the user may see a conflict in library code they don't expect to modify.

WESL library publishing tools will address this by expanding wildcards to named
imports in the published version of the module:

```wesl
// source
import bevy::prelude::*;
```

```wesl
// published artifact
import bevy::prelude::{Color, Mesh, Transform, /* snapshot at publish time */};
```

Until WESL packaging tools implement this expansion, library authors should
avoid wildcard imports from external libraries they don't control.

## Import errors and warnings

WESL emits errors for name collisions and for external wildcards from unmarked
modules, and warnings for shadowing that could surprise readers. Genuine
collisions cannot be suppressed; other diagnostics are suppressible via
`@diagnostic`.

| Situation | Behavior |
| --- | --- |
| Local declaration conflicts with named import | Error |
| Named import conflicts with named import | Error |
| Wildcard import conflicts with wildcard import (when name is referenced) | Error |
| Wildcard import from a non-`@wildcardable` external module | Error (`wildcard_import`); suppressible |
| Local declaration or named import shadows a wildcard-imported name | Warning (`wildcard_shadow`); suppressible |
| Wildcard import shadows a WGSL builtin | Warning (`builtin_shadow`); suppressible |

When multiple wildcard imports are in scope, the same name may be exported by
more than one module:

```wesl
import foo::*; // exports clashing_zap
import bar::*; // exports clashing_zap
```

If `foo::clashing_zap` and `bar::clashing_zap` re-export the same item, there's
no conflict. Otherwise the potential conflict is dormant unless `clashing_zap`
is referenced.

### Scope precedence

When a name could resolve to items at multiple precedence levels, the
highest-precedence one wins. The table above describes whether such a resolution
is silent, produces a warning, or produces an error.

1. user declarations
2. named imports (non-wildcard)
3. wildcard-imported names
4. package names
5. predeclared items (WGSL builtins)

Package names can shadow predeclared items, but we recommend that authors avoid
doing that.

## Forward compatibility with WGSL predeclared identifiers

Predeclared items have the lowest priority in scope resolution (see
[Scope precedence](#scope-precedence)). WGSL can add new predeclared items in
future spec revisions without breaking existing WESL code: any name already
bound by a user declaration, named import, wildcard import, or package continues
to resolve as before.

## Directives
Under discussion, see: <https://github.com/wgsl-tooling-wg/wesl-spec/issues/71>

## Entry points and pipeline overridable constants
These items are preserved when importing a module. Their name must be preserved.
They will land in the final module, if they are statically accessed.

* [Entry points](https://www.w3.org/TR/WGSL/#entry-points)
* [Pipeline overridable constants](https://www.w3.org/TR/WGSL/#override-decls)

For future work, see [@publish GitHub Issue](https://github.com/wgsl-tooling-wg/wesl-spec/issues/65).

## Side-effects and `const_assert`
Generally, WGSL elements are included if they are recursively used from the root module ([statically accessed](https://www.w3.org/TR/WGSL/#statically-accessed)).
An import statement by itself doesn't have any side effects. It does not bring in `const_assert`s.

`const_assert` statements are also included if they are in the same module or namespace as a used element.
This only refers to the exact module that an element is in, and not any of the parent modules.
`let a = bevy_pbr::lighting::shadows::SHADOW_DEPTH;` would bring in the `const_assert`s of the `shadows` module.

Example:

```wgsl
​​​​// main.wesl:
​​​​import foo::bar;
​​​​fn main() { bar(); }

​​​​// foo.wesl:
​​​​import zig::zag;
​​​​const_assert(1 > 0); // included in link because bar is used
​​​​fn bar() { }
​​​​fn miz() { zag() }

​​​​// zig.wesl:
​​​​const_assert(2 < 0); // not included in link
​​​​fn zag() { }
```

Example

```wgsl
import foo::bar;

// Only the foo::bar::baz module would bring in its const assertions
const a: u32 = bar::baz::hello; 
```

`const_assert`s inside functions are treated specially! They can get eliminated during dead-code elimination, which is an observable side-effect.
[WESL deviates from WGSL here](https://github.com/wgsl-tooling-wg/wesl-spec/issues/93).

## Name Mangling
See [Name Mangling](./NameMangling.md)

## Dead Code Elimination
Linkers may choose to do dead code elimination, but it is a non-observable implementation detail.

`const_assert` statements inside of functions need special treatment, see relevant section.

## Visibility
Everything is public by default.

Future proposals will introduce visibility (privacy) for items and/or modules.

# Drawbacks
Are there reasons as to why we should not do this?

* This introduces yet another importing syntax that developers have to learn, instead of using a standard syntax.
* To implement the name mangling, one has to parse WGSL code! This is not trivial, and requires a partial WGSL parser.
* Paths in import statements must consist of valid WGSL identifiers, which can be limiting. This limitation could be lifted by allowing arbitrary strings in import paths, but would make the implementation more complex.

# Rationale and alternatives

## Not agreeing on a standard
One major upside of standardizing it is that it becomes practical for language servers to support it.

The usual alternative is that one library, like shaderc, becomes very popular and the standard ends up being "whatever popular library XYZ does".

An open process lets us find a better solution.

## Preprocessor `#include <lighting.wgsl>`
One alternative, which is common in the GLSL and C worlds, is an including mechanism which simply copy-pastes existing code. A major upside is that this is very simple to implement.

One drawback is that importing the same shader multiple times, which can also happen indirectly, does not work without other preprocessor features.

```c
// A.wgsl
#include <lighting.wgsl>
#include <math.wgsl>
```

```c
// lighting.wgsl
#include <math.wgsl>
```

would not work, since anything defined in `math.wgsl` would be imported twice. In C-land, this is solved by using *include guards*.

Another drawback is that using the same name twice is impossible. In C-land, this leads to pseudo-namespaces, where major libraries will prefix all of their functions with a few symbols. An example of this is the Vulkan API `vkBeginCommandBuffer` or `vkCmdDraw`.

A future drawback is that "privacy" or "visibility" becomes very difficult to implement. Everything that is imported is automatically public and easily accessible.
In C-land, the workaround is using header files. In other languages, such as Python, the convention ends up being "anything prefixed with an underscore `_` is private".

## TypeScript-like imports
The Bevy team, with a large shader codebase, had a few wishes

* A short syntax
* Inline usage

## Rust-like imports
To fully copy Rust's importing syntax, one needs something akin to a `mod` statement.
The rules have carefully been architected to imitate the Rust style, while not requiring an explicit `mod` statement.

In Rust, `use foo::bar;` could either map to "import an item called `bar` from `foo.rs`" or it could map to "import the module `foo/bar.rs`". Rust uses the explicit `mod` statement to disambiguate. We instead check for the presence of an item.

## Putting exports in comments
This would have the advantage of letting some existing WGSL tools ignore the new syntax. For example, a WGSL formatter would not need to know about imports, and could just format the code as usual.

## Using an alternative shader language
There are multiple higher level shading languages, such as [slang](https://github.com/shader-slang/slang) or [Rust-GPU](https://github.com/EmbarkStudios/rust-gpu) which support imports. They also support more features that WGSL currently does not offer. For complex projects, this can very much pay off.

The downside is using additional tooling, and dealing with an additional translation layer.
An additional translation layer could lock shader authors out of certain WGSL features.

Also, higher level GPU languages are typically processed at build time,
which precludes using language features to adapt to runtime conditions
like GPU characteristics or user settings.

## Composing shader code as strings at runtime
One alternative is to compose shader code at runtime
by simply joining together strings with WGSL code, perhaps
with some string templating for flexibility.
This has the major downside of not being statically analyzable.
The IDE cannot provide autocompletion,
and a language server cannot check for errors.

A linker that understands imports also typically
composes shader strings, and can link at runtime.
But a linker uses its more sophisticated understanding of WGSL
to drive composition.
For example, a linker can identify imports
that are needed by other imports,
automating shader composition for users.

# Implementation
Implemented in the [JavaScript/TypeScript](https://github.com/wgsl-tooling-wg/wesl-js) and [Rust](https://github.com/wgsl-tooling-wg/wesl-rs) linkers.

# Test cases
Test cases are available on
[GitHub](https://github.com/wgsl-tooling-wg/wesl-testsuite).

# Future possibilities

## Namespaces
We hope that namespaces will be added to WGSL itself. Then, the importing mechanism can be extended to fully support namespaces, for example by treating each file as introducing its own namespace.

## Source maps
We encourage tooling authors to also implement source maps when implementing imports. This aids

* Error Reporting. When Naga or Tint report an error in the generated WGSL code, we want to map the error location back to the WESL code.
* Debugging. Eventually we hope to have a full toolchain of WESL to WGSL to SPIR-V, with source maps at each step. In the end, it should be possible for RenderDoc to show the original WESL code.

## Scoped imports
Allow imports that are only active within one function?
