# Binding Structs

* Status: Draft
* Created: 2026-09-29
* Issue: [#231](https://github.com/webgpu-tools/wesl-spec/issues/231)

# Overview

This proposal allows for bindings to be explicitly passed to an entrypoint. 
To achieve this, it lets structs contain pointers to bindings. 
These structs contain the entire definition of bindings, 
are provided to the entrypoint and can be passed to further functions.

## Motivation

When reading WGSL code, it is difficult to tell which bindings are *intended to be used* by an entrypoint. 
This is difficult for humans. It also is difficult for tools which want to know what entrypoints can use the same layout.

This is a concrete issue for `'auto'` layouts, which currently cannot be shared.

## Example

```wesl
struct SamplerTexture {
  @group(0) @binding(0) sampler: ...;
  @group(1) @binding(0) texture0: ...;
}
struct ParticlesBindGroup {
  @binding(1) particlesA : ptr<storage, Particles>,
  @binding(2) particlesB : ptr<storage, Particles, read_write>,
}
struct Bindings {
  st0: SamplerTexture0, // groups 0 and 1 set inside
  @group(2) particles: ParticlesBindGroup, // group 2 set outside
}

@compute @workgroup_size(64)
fn main(bindings: Bindings) {
  someFunction(bindings.st0);
  otherFunction(bindings.particles);
}
```

## Binding Structs

Binding structs naturally extend what types can be passed to an entrypoint.

WGSL allows for structs with builtins and user-defined inputs. 
This extends the structs to also be able to contain `ptr`, sampler and texture types.
Structs which contain such types are called 'binding structs'.
Binding structs are allowed to contain further binding structs.

```wgsl
struct MyInputs {
  @location(0) x: vec4<f32>,
  @builtin(front_facing) y: bool,
  
  @group(0) @binding(1) particlesA : ptr<storage, Particles>,
  @group(0) @binding(2) particlesB : ptr<storage, Particles, read_write>,
}
```

Binding structs cannot be constructed in user code. Instead they are provided to the entrypoint.

Binding structs can be passed to functions. This works best when combined with the [unrestricted_pointer_parameters language feature](https://www.w3.org/TR/WGSL/#language_extension-unrestricted_pointer_parameters). 

<!-- 
- readability, only the entrypoint needs to be read to understand which bindings are used.
- no idea if aliasing should be allowed -->

### Restrictions on global bindings

When an entrypoint uses binding structs, it opts in into the binding struct model. 

When a global binding is used, a warning is emitted. 
This gives tools that rely on binding structs a stronger guarantee.
It also guides the user towards only using the bindings that were *intended* for the entrypoint.
Finally, it empowers users of libraries, because one is warned when a libraries internally uses another binding.
A library internally using a binding is usually not intentional and leaks out into the WebGPU host interface.

## Implementation

This feature is implemented by a straightforward desugaring.

Before desugaring, the following rules are validated

- Binding structs cannot be constructed in WGSL.
- An entrypoint that uses binding structs does not use any other bindings.
- Binding structs are allowed to have at most one `ptr<immediate, T>`. This rule is applied recursively.
- In a binding structs, each pair of (binding group, binding number) must be unique. This rule is applied recursively.

We go over each entrypoint. 
For each entrypoint, we go over the parameters.

If the parameter's type is a binding struct, then we need to generate the global bindings. 
A `@group()` attribute is allowed before such parameters. It is interpreted as being before each field which refers to a binding, recursively. It can be overridden by another `@group()` attribute at a deeper layer.

To generate the global bindings, we need to replace the fields that refer to bindings with global bindings.
[sampler](https://www.w3.org/TR/WGSL/#sampler-types) and [texture type](https://www.w3.org/TR/WGSL/#texture-types) fields are translated to a `var generated_name: field_type` binding.
`ptr<address_space, T, access_mode>` fields are translated to a `var<address_space, access_mode> generated_name: field_type` binding.
`ptr<address_space, T>` fields are translated to a `var<address_space> generated_name: field_type` binding.
Attributes are preserved during translations.
Fields with a type that is another binding struct lead to this procedure being applied recursively. The `@group` attribute is allowed before such fields.
The remaining fields are left untouched. 

In each entrypoint, field accesses are then rewritten to use these global bindings.

After translating all entrypoints, we need to translate every function.
For each function parameter with a binding struct type, we need to pass along the fields that were replaced.
Each replaced field is turned into an explicit parameter.
The field accesses are rewritten to use the explicit parameters.
And each function call is rewritten to explicitly pass those parameters.

At the end, empty structs are removed to comply with WGSLs restriction of no empty types. This affects both the generated structs and the function parameters.

> [!NOTE]
> This desugaring mostly assumes `unrestricted_pointer_parameters`.
> It is equally possible to globally trace each unique reference to binding struct fields,
> and rewrite them to refer to the generated global binding.
> For usages with multiple binding structs this requires actually tracing the usages.
> A call to `my_func(some_bindings)` and `my_func(more_bindings)` can have the same type but refer to different bindings.
> To deal with this case, `my_func` either needs to be monomorphized or inlined. 

### Example

```wgsl
// A binding struct
struct MyBindings {
  @builtin(front_facing) 
  y: bool,
  @group(0) @binding(0)
  lights: ptr<storage, array<vec4f, 8>, read>,
  @group(0) @binding(1)
  linear_sampler: sampler,
}

@compute
fn foo(a: MyBindings) {
  let ambient = a.lights[0];
  bar(a);
}

fn bar(a: MyBindings) {
  let light_1 = a.lights[1];
  let s = a.linear_sampler;
}
```

turns into

```wgsl
// Global bindings
@group(0) @binding(0)
var<storage, read> MyBindings_lights: array<vec4f, 8>;
@group(0) @binding(1)
var MyBindings_linear_sampler: sampler;

@compute
fn foo(a: MyBindings) {
  // Inline the usages
  let ambient = MyBindings_lights[0];
  bar(a, MyBindings_lights, MyBindings_linear_sampler);
}

// Explicitly pass along the translated parameters
fn bar(a: MyBindings, a_0: ptr<storage, array<vec4f, 8>, read>, a_1: sampler) {
  let light_1 = a_0[1];
  let s = a_1;
}
```

## TODO: Constrained Variants

There are some constraints which lead to simpler implementations, but worse usability.

We could disallow the `@group(2)` before fields that have a struct type.
This results in a binding structs generation algorithm that only looks at individual structs, instead of the recursive walk.

The two constraints of
1. Each pair of a group and binding number must be unique.
1. `@group` is not allowed before a field that has a struct type.

combine to make the `unrestricted_pointer_parameters` special case go away. Every struct type is now completely unique for a given entrypoint.

## Design considerations

This feature allows for much better reflection based codegen, as it gives a *name* to a set of bindings.
See [#231](https://github.com/webgpu-tools/wesl-spec/issues/231)

## Future Extensions

When this becomes a WebGPU proposal, then `layout: 'auto'` should be updated to take advantage of this.
When two entrypoints use the same binding struct, then `layout: 'auto'` will return compatible layouts.
This works better in a model where entrypoints do not use any global bindings.

### Aliasing

In theory this kind of aliasing would be valid, as long as the desugaring deduplicates it.

```wgsl
struct SamplerTexture0 {
  @group(0) @binding(0) sampler: ...;
  @group(1) @binding(0) texture0: ...;
}
struct SamplerTexture1 {
  @group(0) @binding(0) sampler: ...; // sampler is from the same bind point!
  @group(1) @binding(1) texture1: ...; // but use a different texture
}
struct Bindings {
  st0: SamplerTexture0, // groups 0 and 1 set inside
  st1: SamplerTexture1, // groups 0 and 1 set inside
}
```

However, this comes with certain downsides
- Developers are more likely to run into aliasing restrictions.
- Code generators, one of the big motivations behind this proposal, would struggle to generate a good interface for this. The idea for code generators is that they should be able to provide a maximally struct-like interface on the host side. Calling a shader should feel like passing a few parameters to a function, but the function happens to run on the GPU. Allowing aliasing would mean that the generated interface enforces that each parameter that refers to the same binding actually has been provided with the same binding.
- (For WESL, this aliasing would mean that we have to implement const eval. This is just an engineering problem, not a language design one.)

Instead of allowing aliasing, we propose to make it possible to *construct binding structs* in WGSL. 
This can be done in a future extension.

### Pipeline overridable constants

A future proposal can give the same treatment to pipeline overridable constants.
A useful piece of inspiration is https://www.sebastianaaltonen.com/blog/no-graphics-api#:~:text=Static%20constants