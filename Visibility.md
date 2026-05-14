# Visibility

Visibility controls which other modules can reference or re-export an item, and
which shader-interface declarations WebGPU pipeline creation APIs can use.

| Visibility | Cross-module access | Re-exportable | Pipeline-visible (root module) |
| --- | --- | --- | --- |
| `@public` | Any package | Yes | Yes |
| *package* (default) | Same package only | No | Yes |
| `@private` | Declaring module only | No | No |

## Module visibility

Every WESL item has one of three visibility levels:

* `@public`: visible from any package.
* *package*: visible from any module in the same package.
* `@private`: visible only within the declaring module.

A module can use an item from another module (named through `import`, or through
an inline path such as `other_pkg::math::dot2`) only if that item is visible to
it.

### Syntax

Visibility is set with a `@public` or `@private` attribute on the declaration:

```wesl
fn helper() { ... }                              // package (default)
@public fn dot2(...) -> f32 { ... }              // public
@private const scratch_size: u32 = 64;           // private
```

`@public` and `@private` are WGSL attributes, so the grammar accepts them in any
position among a declaration's other attributes. By convention, place the
visibility attribute last, immediately before the declaration keyword:

```wesl
@group(0) @binding(0) @public var<storage> data: array<f32>;
```

There is no `@package` attribute: an item with neither `@public` nor `@private`
is *package*-visible. (*package* is written in italics here as a level name, not
an attribute name.) A declaration may carry at most one visibility attribute.

### Grammar

WESL adds two attributes to WGSL's attribute set:

```ebnf
visibility_attribute:
| '@' 'public'
| '@' 'private'
```

`@public` and `@private` extend WGSL's `attribute` non-terminal and follow the
same syntax and any-order placement as other attributes. They are accepted on
the global declaration forms WGSL accepts attributes on (`global_variable_decl`,
`global_value_decl`, `function_decl`, `struct_decl`, `type_alias_decl`); a
`const_assert_statement` declares no name and accepts no visibility attribute.
At most one visibility attribute may appear on a single declaration.

See the
[WGSL recursive descent grammar](https://www.w3.org/TR/WGSL/#grammar-recursive-descent)
for WGSL's full attribute production.

### Referring to less-visible declarations

Visibility governs where a declaration can be named, not which other
declarations it may mention. An `@public` declaration may name a less-visible
one: an `@public fn` can return or accept a *package* struct, and an `@public
struct` can have a field of a less-visible type. A consumer that cannot see the
type can still use a value of it (call the function, read its fields) but cannot
name the less-visible type to declare a variable, import it, or construct a new
value.

WESL publishing tools should warn when an `@public` declaration mentions a
less-visible type in its signature or fields; the warning is suppressible with
`@diagnostic(off, leaked_type)` on the declaration.

## Re-exports

`@public import` re-exports the imported names under the current module's path,
in addition to bringing them into local scope.

```wesl
// my_lib/prelude.wesl
@wildcardable module;   // lets external importers use `prelude::*`

@public import super::math::{dot2, cross2};
@public import super::geom::*;
@public import super::types::Mesh as M;
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
Re-exporting one item under several aliases (`@public import super::types::Mesh
as MeshA;` alongside `... as MeshB;`) just adds reachable names; all of them
resolve to the single declaration.

### Only public items can be re-exported

A `@public import` re-exports a name at `@public` visibility. The original
declaration must already be `@public`; `@public import` of a *package* or
`@private` item is an error.

```wesl
// my_lib/internal.wesl
fn helper() { ... }                        // package (default)

// my_lib/prelude.wesl
@public import super::internal::helper;    // error: helper is package
```

### Wildcard re-exports

`@public import <path>::*` follows the same `@wildcardable` rule as plain
wildcard imports:

* Within the current package: always permitted.
* From an external package: permitted when the target module is annotated
  `@wildcardable`. From a non-`@wildcardable` external module, it is a
  suppressible `wildcard_import` error.

## Pipeline visibility

WESL translation always starts from a single root module, which defines the
entire pipeline-visible API: the shader declarations available to WebGPU
pipeline creation APIs. Only three kinds of shader declarations can become
pipeline-visible in `createRenderPipeline` and `createComputePipeline` calls:
**entry points**, **resource variables**, and **pipeline-overridable
constants**.

A pipeline-relevant declaration is in the pipeline-visible API when the root
module declares it with `@public` or *package* visibility, or when the root
module `@public import`s it from another module. A bare `import` in the root
module brings a declaration into local scope but does not add it to the
pipeline-visible API.

A pipeline-relevant item
[statically accessed](https://www.w3.org/TR/WGSL/#statically-accessed) from the
pipeline-visible API's dependency graph but absent from the pipeline-visible API
is a link error (except for overrides with a default value; see below). The
check starts from pipeline-visible declarations and follows references
transitively.

The `import` statement itself does not count as a static access; the imported
item counts as statically accessed when an identifier referencing it appears in
a body, type, initializer, or attribute.

Items declared in the root file keep their root-namespace name in the linked
WGSL output. A non-renamed `@public import` exposes the item under its original
name; a renamed re-import (`@public import some_pkg::pbr_fragment as my_frag;`)
appears as `my_frag` in the pipeline-visible API.

Libraries cannot directly add to the pipeline-visible API. Instead, libraries
publish `@public` declarations for the root module to `@public import`.

### Entry points

An entry point is a function with `@vertex`, `@fragment`, or `@compute`. An
entry point is in the pipeline-visible API only if the root module declares it
or `@public import`s it. Non-entry helper functions are never selectable by
WebGPU pipeline creation merely because they are `@public` or *package*-visible.

```wesl
// pbr_lib/passes.wesl
@fragment @public
fn pbr_fragment() -> @location(0) vec4f { ... }

// app/main.wesl  (root)
@public import pbr_lib::passes::pbr_fragment;    // host can select this entry point
```

### Resource variables

A [*resource variable*](https://www.w3.org/TR/WGSL/#resource-interface) is a
`@group/@binding var<...>` declaration that lets host code provide values to the
shader (uniforms, storage buffers, textures, samplers). A resource variable is
in the pipeline-visible API only if the root module declares it or
`@public import`s it. Resource variables have no host-side fallback, so a
*resource variable* statically accessed but absent from the pipeline-visible API
is a link error.

```wesl
// filter_wgsl/package.wesl
@group(3) @binding(0) @public var<storage, read_write> data: array<f32>;

@compute @workgroup_size(64) @public
fn blur(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = ...;
}

// app/main.wesl  (root)
@public import filter_wgsl::blur;
// error: resource variable `filter_wgsl::data` is used but absent from the pipeline-visible API
// fix: add `@public import filter_wgsl::data;` to the root module
```

The re-import does not create a new resource variable; the original
`@group(3) @binding(0)` annotations carry through unchanged.

### Pipeline-overridable constants

A [*pipeline-overridable constant*](https://www.w3.org/TR/WGSL/#override-decls)
is an `override` declaration the host may set at pipeline creation. An override
is in the pipeline-visible API only if the root module declares it or
`@public import`s it.

An override with a default value that is absent from the pipeline-visible API
silently degrades to a constant: the linker bakes in the default and the host
cannot set it. An override without a default value has no such fallback, so an
override statically accessed but absent from the pipeline-visible API is a link
error.

```wesl
// pbr_lib/lighting.wesl
@public override sun_intensity: f32 = 1.0;       // has default
@public override max_lights: u32;                // no default
@public fn apply_lighting() -> vec4f { ... }     // uses both overrides

// app/main.wesl  (root)
import pbr_lib::lighting::apply_lighting;
@public import pbr_lib::lighting::max_lights;    // host must set

@fragment
fn fragment_main() -> @location(0) vec4f {
  return apply_lighting();
}

// sun_intensity not in the pipeline-visible API: linker bakes in 1.0
```

### Wildcard re-export at root

A wildcard `@public import path::*` in the root module brings every
re-exportable top-level item from the target module into the root namespace at
once. The pipeline-relevant items among them (entry points, resource variables,
overrides) become pipeline-visible, a convenience for external libraries that
ship a curated bundle. Wildcard re-export at root follows the same
`@wildcardable` rule as any other wildcard import (see
[Wildcard re-exports](#wildcard-re-exports)).

```wesl
// app/main.wesl  (root)
@public import pbr_lib::resources::*;   // external library's curated resource bundle
```

### Aggregating entry points

When an app's entry points live in multiple source files, a small root module
brings them together. The entry points must be declared `@public` so the root
can re-export them (see
[Only public items can be re-exported](#only-public-items-can-be-re-exported)):

```wesl
// app/vertex.wesl
@vertex @public fn vertex_main(...) -> @builtin(position) vec4f { ... }

// app/fragment.wesl
@fragment @public fn fragment_main(...) -> @location(0) vec4f { ... }

// app/main.wesl  (root)
@public import super::vertex::vertex_main;
@public import super::fragment::fragment_main;
```
