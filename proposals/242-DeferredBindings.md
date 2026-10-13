# Deferred Bindings (experimental)

Deferred bindings let you write WESL shaders 
without choosing `@group` and `@binding` numbers for resources like buffers and textures.
A system API arranges groups and bindings, and returns `GPUBindGroupLayout` objects by name.
Libraries and many applications will never need to see the group and binding numbers at all.
(The group and binding numbers are available via reflection for more advanced uses.)

[wgsl-play.dev example](https://wgsl-play.dev/#v1=N4IgLglmA2CmIC4QBNYDNYCdO2QWgCMIA7ZEgcwGcQAaEAB0wHsArWAYzEVAHdZLoAZUztuDAIbsA1uPKwECALbiSiEDgCOAVwg5KAAj4CA+qgzZcxoqQqUA3AB1iTgG7jMAHkpgmmWbBp9HHFkYx5MKFgAPn1kcTBxBH13PwBPDzQAZgAmQIAWKLt9AHpi-XJmLXpk0n1rMmJyZMpKCHJiXDrU-TAAC1h9SlTvWEUnJwABdiZFei0wAYmeXykKpirjVoAvWAAKPIBKJzRifWUSXYmCHWhIYl3yaCYCcWhjEhcmdniIJmJ35AHfQQZBJFwcTJaIHAJz6WLxcQAbRBADoAB4AXX0AF59FlsrtUWigQBqfQARhRAAZHMQAL5OEB0ujMJhgACyTGQWjgADlxIp4EhzsQmXSgA):
```wgsl
requires wesl_deferred_bindings;

var<storage, read_write> data: array<f32, 4>; // group and binding assigned by the system

@compute @workgroup_size(4)
fn main(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = f32(id.x) + 1.0;
}
```

Load this shader in wgsl-play to see `data` in the results panel,
or pass it to wgsl-test's `testCompute()` to get `{ data: [1, 2, 3, 4] }`.
wgsl-play's [`readVars()`](README.md#reading-shader-state) also reads storage
buffers by name.

| feature level | enables additional feature
| --- | --- |
| Today | manually choose binding numbers, "auto" layouts are not recommended
| + deferred bindings | applications bind resources by name, binding structs for "auto" consistency, libraries don't hard code numbers
| + do blocks | resource allocation in do code, pass bound resources and binding structs to entry points
| + wgsl-play/test | automatic creation of global resources like buffers and textures

## Contents

- [Group and binding numbers](#group-and-binding-numbers)
- [Binding structs](#binding-structs)
  - [Creating a binding struct in a do block](#creating-a-binding-struct-in-a-do-block)
  - [Fields and arguments](#fields-and-arguments)
  - [Groups and binding numbers](#groups-and-binding-numbers)
  - [Passing fields onward](#passing-fields-onward)
- [Deferred bindings in wgsl-play and wgsl-test](#deferred-bindings-in-wgsl-play-and-wgsl-test)
- [Implementation notes](#implementation-notes)
- [Future work](#future-work)
  - [Reflection by name](#reflection-by-name)

## Group and binding numbers

The system puts resources without numbers in the lowest available group (typically group 0)
and uses the lowest binding numbers not already in use.

| Declaration | Behavior |
| --- | --- |
| Neither `@group` nor `@binding` | The system assigns group and binding numbers. |
| `@group(g) @binding(n)` | Keeps both. The system skips `n` when `g` is 0. |
| `@binding(n)` alone | Keeps `n`. The system adds `@group(0)` and skips `n`. |
| `@group(g)` alone | Error: add a binding number or remove the group. |

## Binding structs

A **binding struct** groups resources into a bind group. The entry-point
parameter's `@group(n)` sets the group. 

Writing binding structs lets you share bind groups between pipelines, which
is good for performance. Binding structs also fix an instability in current
WebGPU's `layout: "auto"`, where bind group layouts change as you edit the
WGSL (comment out the last use of a resource and the layout changes).
A binding struct declares the layout explicitly instead.
So we get the "fully explicit, non-exclusive auto layouts" that
[gpuweb#4957](https://github.com/gpuweb/gpuweb/issues/4957) wants.

### Creating a binding struct in a do block

Declare storage pointer fields and use a WGSL-style struct constructor to
choose the buffers. Pass the constructed value to the entry point:

[wgsl-play.dev example](https://wgsl-play.dev/#v1=N4IgDgTg9gVgpgYwC4gFygO5wM4BsDKECaoYAhggNZkDmcqqAtmQJYB2aIEcAjgK4tu2AARY8AfQAmUcQCNcUKtgA0onLilwAZnAjdJc9pPY1sAbgA6bKwDcyEADzYkUCLTiruZAxggskcAB8wmBQ2P4sUGzYqML2bgCeDjaIACxaqgBsqYGWbHaOzq7unnDe4r7+QcIpCggROLHxZEkpCOlZOXlWzhB8yMIACvZILAi4cABCRiYiwFbCIWGxYEiFLm50qs2taVqBpeWVAQfCAPRnwrIzbDTCAAwLNXC4K2tOGyVxei3Je4GnRZA4FAi5XG53ACMh0kAFoorgElYAL5WKzSYR9NgACgAlMJ5mxFgABYzYchIBAAC2xkL4+OwcDgkjxeUWEyQIRGYwmIgAvENueMphDsNiAGShcKjKIqYTi2qKBrYXFs4SkljksiUml0-HeOxsBBwbHkCCjYUqtUarU62n0uKSQ3G01C3mqlFotjEhBQRhgPgBdUYVyUGjQPhgcThABeJuyuKsWiJjOZ2OJsgEuFGOJoClkZA07Bsim1kTY4hYknxVdibQAzA7CYtFfVRjgANpVgB0AA8ALrCAVtdLYrT1gBM2J7vdxqkh3fuqnui+Xi49bFR1m9vv9gbgwdD4agkejLDj2ITSaJBrIRpNxOPkdp+LNFt5sWG5p5IrYxlucpgk+YDCJCTwgsCGZZjm2J5lABZFvkpYyhWVY1pIdaII2+LNly36Wt2UpdpIfaDgA1AKb4-tg3a1MRpF5MiIDIsoXBQFASAALJQJIfATAAcmQjBwJwzDsCArG+n+ESyiQyIsSAoxIBMnDXNJtywr0-RINgzFAA).
Select 'View > Compiled' to see the lowered WGSL.

```wgsl
requires wesl_do_blocks, wesl_deferred_bindings;

var<storage, read_write> positions: array<vec4f, 64>;
var<storage, read_write> velocities: array<vec4f, 64>;

struct ParticleBindings {
  pos: ptr<storage, array<vec4f>, read_write>, // binding 0
  vel: ptr<storage, array<vec4f>>,             // binding 1, read-only
}

do run() {
  @dispatch(1u) seed();
  let particles = ParticleBindings(&positions, &velocities);
  @dispatch(1u) advance(particles);
  @dispatch(1u) advance(particles);
}

@compute @workgroup_size(64)
fn seed(@builtin(global_invocation_id) id: vec3u) {
  velocities[id.x] = vec4f(f32(id.x), 1.0, 0.0, 0.0);
}

@compute @workgroup_size(64)
fn advance(@group(1) particles: ParticleBindings, // group 1
           @builtin(global_invocation_id) id: vec3u) {
  particles.pos[id.x] += particles.vel[id.x];
}
```

The two dispatches to `advance` reuse the same bind group. 
`seed` writes the same `velocities` through the module-scope var;
`advance` reads it read-only through `particles.vel`.

### Fields and arguments

Binding struct fields are storage buffer pointers, `ptr<storage, T, access>`.

The binding struct constructor works like a WGSL struct constructor,
one argument per field, in field declaration order.

A fixed length array field can be set with a pointer to a buffer with more elements;
the extra elements are ignored.

A `read_write` field requires a writable buffer. A `read` field accepts
either access mode. 

Binding struct fields are immutable, they cannot be changed after construction.

The constructed binding struct can be held in a do block `let` or `var`, 
or in a module-scope `do var` to reuse the bind group across runs.

Plain values such as `u32` are not allowed in a binding struct. Pass them as
[per-dispatch values](DoBlocks.md#value-parameters) instead.

### Groups and binding numbers

An entry point that takes a binding struct must annotate that parameter
with `@group(n)` for now, as `advance` does above.

A binding struct field with `@binding(n)` uses that number in the group.
Other fields take the lowest free binding numbers in declaration order.
Fields can't have `@group`, the group comes from the parameter.

A binding struct describes one bind group. Entry points that take the same
struct share its bind group layout. Two binding structs can't have the same group.
A module-scope variable can't declare a binding in a binding
struct's group.

### Passing fields onward

An entry point can pass a binding struct field (e.g. `particles.pos`) to a
shader function with a pointer parameter. This requires
`unrestricted_pointer_parameters`, which Firefox does not support.

Do code can pass a field to another entry point's buffer parameter directly,
e.g. `@dispatch(1u) scale(particles.pos)`, with no `&` and no language
feature needed.

## Deferred bindings in wgsl-play and wgsl-test

[wgsl-play](README.md) and [wgsl-test](../wgsl-test) create and bind 
buffers for module scope resources without `@group` nor `@binding` attributes.
Storage buffers may be `read` or `read_write`, and are
zero-filled at initialization. A buffer without binding numbers needs a
fixed size so that the system can allocate it. Give a runtime-sized array,
such as `array<vec4f>`, a fixed length, or use
[`createBuffer<T>(n)`](DoBlocks.md#allocate-buffers-with-createbuffertn)
in a do block instead.

Textures and samplers will follow the same rules. For now, a texture or
sampler without binding numbers is handled by wgsl-play's
[`@texture(name)` and `@sampler(filter)`](README.md#resources) annotations,
wgsl-test's [`@test_texture` and `@sampler`](../wgsl-test/README.md#test-resources),
or a resource passed to wgsl-test's
[`testFragment()`](../wgsl-test/API.md#testfragment).

## Implementation notes

Before linking, the system replaces each binding struct parameter with
module-scope variables, one per field, in the struct's group. The lowered
WGSL has explicit group and binding numbers written in. References
such as `particles.pos` become pointers to those variables.

Resources without binding numbers are numbered in order of their names,
compared by Unicode code point.

wgsl-play and wgsl-test also use deferred bindings internally for their own
needs, e.g. for the uniform buffer behind wgsl-play's `@uniforms` struct.

The `@buffer` annotation is now retired.

## Future work

- wgsl-test's `testCompute()` currently fills buffers with -999,
  but should likely move to zero filling for consistency.
- Binding struct fields for uniform buffers (`ptr<uniform, T>`), filled from
  a do block.
- Binding struct fields for textures and samplers.
- Binding structs in group 0. Today a binding struct needs a group other
  than 0, which holds the resources without binding numbers.
- Binding struct parameters without group annotations.
- A binding struct taken at different groups by different entry points.
  (Today every entry point that takes a struct must use the same group.)
- Binding structs declared in a different module from their entry points.
  Today the struct and every entry point that takes it share a module, and
  if that module is imported, the main module must also import or refer to
  the struct.
- Passing a whole binding struct to a non entry point shader function.
  Today an entry point passes the fields separately to other shader functions.
- Nested binding structs: a field whose type is another binding struct, laid
  out in the enclosing struct's group and passable to shader functions on its
  own. Today every field must be a pointer.
- A lone `@group(g)` that places a resource in group `g` while the system
  picks its binding number. Today a lone `@group(g)` is an error, and the
  system puts every resource without numbers in group 0.
- A JavaScript reflection API for reading assigned numbers and layouts by
  name, sketched below.
- A polyfill so that plain WebGPU `layout: "auto"` derives layouts from
  binding structs.

### Reflection by name

A binding struct maps directly to a bind group layout, so the struct's
name identifies the layout. Resources without numbers are found by their names too. 

The name based API sketched below lets host code use those layouts without
seeing a number:

```js
// layouts by name: the binding struct from the example above
const particlesLayout = module.getBindGroupLayout("ParticleBindings");

// layouts w/o binding structs under the reserved name "package" 
const looseLayout = module.getBindGroupLayout("package");

// bind groups with entries by name with the bind group layout
const particles = device.createBindGroup({
  layout: particlesLayout,
  entries: { 'pos': posBuffer, 'vel': velBuffer }, 
});
const loose = device.createBindGroup({
  layout: looseLayout,
  entries: { 'data': dataBuffer }, // 'data' from the first example
});

pass.setBindGroup(particles);
pass.setBindGroup(loose);
```

Reflection reports the names in the shader code and details about binding
numbers for more advanced applications.

```js
const info = await module.getReflection();

// every bind group by name, the loose one under "package"
Object.keys(info.bindGroups); // -> ["ParticleBindings", "package"]

info.bindGroups.ParticleBindings.entries;
// -> [{ name: "pos", binding: 0, buffer: { type: "storage" } },   // runtime-sized array, no size
//     { name: "vel", binding: 1, buffer: { type: "read-only-storage" } }]
info.bindGroups.package.entries;
// -> [{ name: "data", binding: 0, buffer: { type: "storage" }, size: 16 }]

info.entryPoints.main.bindGroups; // -> ["package", "ParticleBindings"]

// assigned numbers, for host apps that want the numbers
info.bindGroups.ParticleBindings.group; // -> 1
```

The names are the ones the main WESL module uses (including e.g. `as` renames).

In wesl-js's internal wesl-gpu library, `bindingStructLayout()` 
and `bindingStructBindGroup()` do most of the necessary name based lookup 
and binding work, but the user reflection API here is not implemented.
