# Visibility

Visibility controls which other modules can reference or re-export an item. For
entry points, resource variables, and pipeline-overridable constants, it also
controls which items the root module exposes to the host.

Every WESL item has one of three visibility levels:

* `public`: visible from any package.
* *package* (the default): visible from any module in the same package.
* `private`: visible only within the declaring module.

| Visibility | Cross-module access | [Re-exportable](#a-re-export-cannot-widen-visibility) | Root-declared [pipeline-relevant item](#pipeline-visibility) |
| --- | --- | --- | --- |
| `public` | Any package | Yes | Pipeline-visible |
| *package* (default) | Same package only | No | Pipeline-visible |
| `private` | Declaring module only | No | Not pipeline-visible |

## Visibility between modules

A module can use an item from another module (named through `import`, or through
an inline path such as `other_pkg::math::dot2`) only if that item is visible to
it. A wildcard import omits items that are not visible to the importer. A name
collision between two wildcard imports is therefore an error only when both
items are visible (see [Wildcard imports](Imports.md#wildcard-imports)).

### Syntax

A visibility keyword may prefix any item declaration, between any attributes
and the declaration keyword:

```wesl
fn helper() { ... }                              // package (default)
public fn dot2(...) -> f32 { ... }               // public
private const scratch_size: u32 = 64;            // private

@group(0) @binding(0) public var<storage> data: array<f32>;
```

There is no `package` visibility keyword: an item with no `public` or `private`
keyword is *package*-visible. (*package* is written in italics here as a level
name, not a keyword like `public` or `private`.)

`private` is a context-dependent name in WGSL. WESL treats it as a visibility
keyword only in this declaration position, and it remains available as an
identifier elsewhere.

### Grammar

Visibility extends WGSL's global declaration forms with an optional `public` or
`private` keyword between any attributes and the leading declaration keyword.

```ebnf
visibility:
| 'public' | 'private'

global_variable_decl: | attribute* visibility? ...
global_value_decl:    | attribute* visibility? ...
function_decl:        | attribute* visibility? ...
struct_decl:          | attribute* visibility? ...
type_alias_decl:      | attribute* visibility? ...
```

`...` stands for the rest of each WGSL form, unchanged; see the
[WGSL recursive descent grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent)
for the full productions. `const_assert_statement` takes no visibility because
it declares no name.

### Referring to less-visible declarations

Visibility governs where an item can be referenced or imported, not how
visible its referrers must be. A `public` declaration may refer to a less-visible
one: a `public fn` can return or accept a *package* struct, and a `public struct`
can have a field of a less-visible type. A consumer that cannot see the type can
still use a value of it (call the function, read its fields) but cannot name the
type to declare a variable, import it, or construct a new value.

Naming a less-visible type in a `public` signature or field is permitted with no
diagnostic.

## Re-exports

`public import` re-exports the imported names under the current module's path,
in addition to bringing them into local scope. It re-exports individual items,
not whole modules.

```wesl
// my_lib/prelude.wesl
@!wildcardable;   // lets external importers use `prelude::*`

public import super::math::{dot2, cross2};
public import super::types::Mesh as M;
```

After this, importers can write:

```wesl
import my_lib::prelude::dot2;
import my_lib::prelude::M;
import my_lib::prelude::*;       // wildcard pulls all of the above
```

A bare `import` brings names into local scope without re-exporting them.

### Re-export identity

Re-exports add reachable paths but do not create new declarations. An item's
identity is its declaration path at the definition site: the package name, the
chain of module names to the declaration site, and the declared name (for
example `my_lib::types::Mesh`). A re-export's reachable name (the last segment
of the re-export path, or its `as` alias) can differ from the declared name, but
identity follows the original declaration. Two paths that resolve to the same
identity collapse with no name collision and no duplicated emission.
Re-exporting one item under several aliases (`public import super::types::Mesh
as MeshA;` alongside `... as MeshB;`) just adds reachable names; all of them
resolve to the single declaration.

### A re-export cannot widen visibility

A `public import` re-exports a name at `public` visibility. The original
declaration must already be `public`; re-exporting a *package* or `private` item
as `public` would widen its visibility, and is an error.

```wesl
// my_lib/internal.wesl
fn helper() { ... }                        // package (default)

// my_lib/prelude.wesl
public import super::internal::helper;    // error: helper is package
```

## Pipeline visibility

WESL translation starts from a single root module, which determines the entire
pipeline-visible API. Only three kinds of items, called *pipeline-relevant
items*, can be pipeline-visible: **entry points**, **resource variables**, and
**pipeline-overridable constants**. The pipeline-visible items form the
host-facing interface of the linked shader.

After [conditional translation](ConditionalTranslation.md), a
pipeline-relevant item is in the pipeline-visible API when the root module
declares it with `public` or *package* visibility, or when the root module
`public import`s it from another module. A bare `import` in the root module
brings an item into local scope but does not add it to the pipeline-visible API.

A resource variable or `override`
[statically accessed](https://www.w3.org/TR/WGSL/#statically-accessed) from the
root module's dependency graph but absent from the pipeline-visible API is a
link error (except for a defaulted `override`; see below). The check starts from
declarations in the root module and follows references transitively.

An `import` statement is not itself a static access. An imported item is
statically accessed when an identifier referring to it appears in a body, type,
initializer, or attribute.

Pipeline-visible items keep their exposed names in the linked WGSL output. A
root-declared item keeps its declared name. An item re-exported by `public
import` keeps its imported name, or its `as` alias if renamed. For example, `public import
some_pkg::pbr_fragment as my_frag;` exposes `my_frag`. Other items may be
mangled by the linker.

Libraries cannot directly add to the pipeline-visible API. Instead, libraries
publish `public` items for the root module to `public import`.

### Entry points

An entry point is a function with `@vertex`, `@fragment`, or `@compute`. An
entry point must be pipeline-visible for WebGPU pipeline creation to select it.
Non-entry helper functions are never selectable merely because they are
`public` or *package*-visible.

```wesl
// pbr_lib/passes.wesl
@fragment
public fn pbr_fragment() -> @location(0) vec4f { ... }

// app/main.wesl  (root)
public import pbr_lib::passes::pbr_fragment;    // host can select this entry point
```

### Resource variables

A [*resource variable*](https://www.w3.org/TR/WGSL/#resource-interface) is a
`@group/@binding var<...>` declaration that lets host code provide values to the
shader (uniforms, storage buffers, textures, samplers). Resource variables have
no host-side fallback, so a resource variable statically accessed but absent
from the pipeline-visible API is a link error.

```wesl
// filter_wgsl/package.wesl
@group(3) @binding(0) public var<storage, read_write> data: array<f32>;

@compute @workgroup_size(64)
public fn blur(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = ...;
}

// app/main.wesl  (root)
public import filter_wgsl::blur;
// error: `filter_wgsl::data` is statically accessed but not pipeline-visible
// fix: add `public import filter_wgsl::data;` to the root module
```

The `public import` does not create a new resource variable; the original
`@group(3) @binding(0)` annotations carry through unchanged.

### Pipeline-overridable constants

A [*pipeline-overridable constant*](https://www.w3.org/TR/WGSL/#override-decls)
is an `override` declaration the host may set at pipeline creation.

A defaulted `override` that is absent from the pipeline-visible API is not
host-settable and takes its default value. A WESL linker may emit it as an
`override` with a mangled name and no `@id`, transform it into a `const` if its
default is a const-expression, or inline its initializer at use sites. These
strategies are observably equivalent to the host. A non-defaulted `override`
absent from the pipeline-visible API is a link error if it is statically
accessed from the root module.

```wesl
// pbr_lib/lighting.wesl
public override sun_intensity: f32 = 1.0;       // has default
public override max_lights: u32;                // no default
public fn apply_lighting() -> vec4f { ... }     // uses both overrides

// app/main.wesl  (root)
import pbr_lib::lighting::apply_lighting;
public import pbr_lib::lighting::max_lights;    // host must set

@fragment
fn fragment_main() -> @location(0) vec4f {
  return apply_lighting();
}

// sun_intensity not in the pipeline-visible API: host cannot set it; uses its default of 1.0
```

### Aggregating entry points

When an app's entry points live in multiple source files, a small root module
brings them together. The entry points must be declared `public` so the root can
re-export them (see
[A re-export cannot widen visibility](#a-re-export-cannot-widen-visibility)):

```wesl
// app/vertex.wesl
@vertex public fn vertex_main(...) -> @builtin(position) vec4f { ... }

// app/fragment.wesl
@fragment public fn fragment_main(...) -> @location(0) vec4f { ... }

// app/main.wesl  (root)
public import super::vertex::vertex_main;
public import super::fragment::fragment_main;
```
