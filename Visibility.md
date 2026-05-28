# Visibility

Visibility controls which other modules can reference or re-export an item, and
which shader-interface declarations WebGPU pipeline creation APIs can use.

| Visibility | Cross-module access | [Re-exportable](#a-re-export-cannot-widen-visibility) | Pipeline-visible (root module) |
| --- | --- | --- | --- |
| `public` | Any package | Yes | Yes |
| *package* (default) | Same package only | No | Yes |
| `private` | Declaring module only | No | No |

## Module visibility

Every WESL item has one of three visibility levels:

* `public`: visible from any package.
* *package*: visible from any module in the same package.
* `private`: visible only within the declaring module.

A module can use an item from another module (named through `import`, or through
an inline path such as `other_pkg::math::dot2`) only if that item is visible to
it. A wildcard import silently skips items that are not visible to the importer,
so a name clash between two wildcard imports is an error only when both items are
visible to the importer (see [Imports.md](Imports.md)).

### Syntax

A visibility keyword may prefix any item declaration, after any attributes:

```wesl
fn helper() { ... }                              // package (default)
public fn dot2(...) -> f32 { ... }               // public
private const scratch_size: u32 = 64;            // private
```

The visibility keyword sits between any attributes and the declaration keyword:

```wesl
@group(0) @binding(0) public var<storage> data: array<f32>;
```

There is no `package` visibility keyword: an item with no `public` or `private`
keyword is *package*-visible. (*package* is written in italics here as a level
name, not a keyword like `public` or `private`.)

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

Visibility governs where a declaration can be named, not which other
declarations it may mention. A `public` declaration may name a less-visible one:
a `public fn` can return or accept a *package* struct, and a `public struct` can
have a field of a less-visible type. A consumer that cannot see the type can
still use a value of it (call the function, read its fields) but cannot name the
less-visible type to declare a variable, import it, or construct a new value.

Naming a less-visible type in a `public` signature or field is permitted with no
diagnostic.

## Re-exports

`public import` re-exports the imported names under the current module's path,
in addition to bringing them into local scope. It re-exports individual items,
not whole modules.

```wesl
// my_lib/prelude.wesl
@wildcardable module;   // lets external importers use `prelude::*`

public import super::math::{dot2, cross2};
public import super::types::Mesh as M;
```

After this, importers can write:

```wesl
import my_lib::prelude::dot2;
import my_lib::prelude::M;
import my_lib::prelude::*;       // wildcard pulls all of the above
```

(Bare `import` brings names into local scope without re-exporting.)

### Re-export identity

Re-exports add reachable paths but do not create new declarations. An item's
identity is its absolute module path: the package name, the chain of module
names to the declaration site, and the declared name (for example
`my_lib::types::Mesh`). A re-export's reachable name (the last segment of the
re-export path, or its `as` alias) can differ from the declared name, but
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

WESL translation always starts from a single root module, which defines the
entire pipeline-visible API: the shader declarations available to WebGPU
pipeline creation APIs. Only three kinds of shader declarations can become
pipeline-visible in `createRenderPipeline` and `createComputePipeline` calls:
**entry points**, **resource variables**, and **pipeline-overridable
constants**.

A pipeline-relevant declaration is in the pipeline-visible API when the root
module declares it with `public` or *package* visibility, or when the root
module `public import`s it from another module, under the current
[conditions](./ConditionalTranslation.md). A bare `import` in the root module
brings a declaration into local scope but does not add it to the
pipeline-visible API.

A pipeline-relevant item
[statically accessed](https://www.w3.org/TR/WGSL/#statically-accessed) from the
root module's dependency graph but absent from the pipeline-visible API is a
link error (except for `override` with a default value; see below). The check
starts from declarations in the root module and follows references transitively.

The `import` statement itself does not count as a static access; the imported
item counts as statically accessed when an identifier referencing it appears in
a body, type, initializer, or attribute.

Pipeline-visible items keep their original name in the linked WGSL output:
root-declared items keep their root-namespace name, and re-exported items keep
their original name (or the `as` alias if renamed, as in `public import
some_pkg::pbr_fragment as my_frag;`). Non-pipeline-visible items may be mangled
by the linker.

Libraries cannot directly add to the pipeline-visible API. Instead, libraries
publish `public` declarations for the root module to `public import`.

### Entry points

An entry point is a function with `@vertex`, `@fragment`, or `@compute`. An
entry point is in the pipeline-visible API only if the root module declares it
or `public import`s it. Non-entry helper functions are never selectable by
WebGPU pipeline creation merely because they are `public` or *package*-visible.

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
shader (uniforms, storage buffers, textures, samplers). A resource variable is
in the pipeline-visible API only if the root module declares it or
`public import`s it. Resource variables have no host-side fallback, so a
*resource variable* statically accessed but absent from the pipeline-visible API
is a link error.

```wesl
// filter_wgsl/package.wesl
@group(3) @binding(0) public var<storage, read_write> data: array<f32>;

@compute @workgroup_size(64)
public fn blur(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = ...;
}

// app/main.wesl  (root)
public import filter_wgsl::blur;
// error: resource variable `filter_wgsl::data` is used but absent from the pipeline-visible API
// fix: add `public import filter_wgsl::data;` to the root module
```

The re-import does not create a new resource variable; the original
`@group(3) @binding(0)` annotations carry through unchanged.

### Pipeline-overridable constants

A [*pipeline-overridable constant*](https://www.w3.org/TR/WGSL/#override-decls)
is an `override` declaration the host may set at pipeline creation. An
`override` is in the pipeline-visible API only if the root module declares it or
`public import`s it.

An `override` that is absent from the pipeline-visible API is not host-settable
and takes its default value. A WESL linker may transpile an `override` with a
default value to an `override` in WGSL output with a mangled name and no `@id`,
transform it into a `const` if its default is a const-expression, or inline its
initializer at use sites. These strategies are observably equivalent to the
host. If the `override` doesn't have a default value, it is a link error if the
`override` is statically referenced from the root module.

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
