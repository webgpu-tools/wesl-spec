# WGSL Modules


* Status: [Draft](README.md#status-draft)
* Created: 2026-09-09
* Issue: [#TODO](https://github.com/gpuweb/gpuweb/issues/TODO)

# Overview

This proposal intends to add the low level building blocks to enable module systems to be built upon WGSL.

## Background

From a user perspective, a module system has two aspects.

1. Declaring a module
```
module FooModule for "./foo.wgsl"; 
```

2. Importing items from a module
```
import Lygia::Math::PerlinNoise;
```

This proposal does not deal with the above.

From an implementation perspective, one needs to answer the following questions
- What are the contents of a module
- How do qualified paths resolve to a module

This proposal is purely concerned with the implementation perspective. It adds the low level building blocks.

## Motivation

With a WGSL module system, the following use-cases become unblocked.

1. Shadowed builtins can be accessed with a qualified path.
```wgsl
// shadowing the min() function
const min = 3;

// qualified path
const bar = std::min(2, 3);
```

2. Composing multiple shaders with namespaces. See [Ruffle](https://github.com/ruffle-rs/ruffle/blob/master/render/wgpu/shaders/common.wgsl)'s code for an example of `prefix__` namespacing in the wild.

```wgsl
const bar = my_shader::bar;
```

3. Enabling the use of qualified path in the WebGPU API. 

4. Providing a compilation target for higher level languages such as WESL.
TODO: Rephrase this as "the module system should be prepared for a future where it is extended to higher level concerns, such as loading from files and import statements"

5. Providing a bundling target.

## Description

Imagine the namespaces proposal being right here https://github.com/gpuweb/gpuweb/pull/7310 . 
Do mentally replace the word `namespace` with `module` to fit in with the rest.

> A module is declared with the `module` keyword. 
> [...]
> [...]


### WebGPU API

The qualified paths (~namespace prefixed) can be used in the WebGPU APIs. 
For example, when creating a compute pipeline, one specifies a path to access items from nested modules.

```ts
const computePipeline = device.createComputePipeline({
  layout: "auto",
  compute: {
    module: shaderModule,
    entryPoint: "render::vs_main",
  },
  constants: {
    "render::lightmapping::SHADOW_SAMPLES": 4,
  },
});
```

### Re-exports

Bigger projects often define types and functions in a nested module and then re-export it higher up.

For example, the Lygia shading library is composed of very small files. 
We want to enable re-exporting the most common Lygia functionalities at a higher level.
For example, the [`lygia::color::space::srgb2rgb::srgb2rgb`](https://github.com/patriciogonzalezvivo/lygia/blob/main/color/space/srgb2rgb.wgsl) 
function is frequently useful and could be re-exported at `lygia::color::srgb2rgb`.

Additionally, we propose *module re-exports* for the needs of bigger projects.

The Bevy game engine is a case study for this. 
The game engine is structured into multiple smaller projects, such as `bevy_pbr` and `bevy_render`.
However, Bevy wants to provide a single coherent view to its users.
The user only installs `bevy` via the Cargo package manager. Then, they import from `bevy::...`. 

To expose `bevy_pbr` and `bevy_render`, Bevy re-export their modules. 
`bevy_pbr` becomes `bevy::pbr` and `bevy_render` becomes `bevy::render`.

One complexity of re-exports is that a single item can be accessed via multiple paths. 
Both `lygia::color::space::srgb2rgb::srgb2rgb` and `lygia::color::srgb2rgb` refer to the same function.
This also means that entrypoints, bindings and overrides can have multiple paths that referring to them.
This directly affects the WebGPU API. Ideally, all paths are valid and can be used.

TODO: Propose a syntax for re-exports!

### Bundling

Multiple files can be bundled into a single string. For example, WESL can bundle a set of shaders and embed this into the final binary.
This enables the Rust use-case of shipping a single executable with everything bundled into it.

## Future work

### Visibility

There are two use-cases that benefit from visibility controls.

The first one are libraries. In ecosystems such as Rust, it is expected that libraries follow semantic versioning and that patch version upgrades should always compile.
To enable this, library authors will carefully curate a public API, for which they promise semantic versioning. Everything else is an implementation detail that is not visible to the user.

The second one is reflection. A WGSL reflection API allows the user to introspect shaders and extract entrypoint names, bind groups and structs.
This can, for example, be used to automatically create type safe wrappers in the host language (TypeScript, Rust, C++).
Not every detail from the shaders is stable and should land in the reflection API. As such, being able to distinguish between public and private is useful.

### High level module system

A very abstract high level module system is described here. The syntax is intended as a minimal placeholder.

A module system has an explicit form of declaring a module.
```wgsl
module foo
```

It allows distinguishing between public and private modules at the point of declaration.
```wgsl
public module foo
private module bar
```

And it offers a way to get the contents of a module.
```wgsl
public module foo for "./foo.wgsl";

public module bar {
    const A = 3;
}
```
The above syntax implies a filesystem, but it does not necessitate one. Paths can be arbitrarily mapped to URLs or to a hashmap lookup.

Finally, once the pieces are in place, it allows for import statements. This enables a shorter syntax than qualified paths.
```
import bar::A;
```

### HMR

Frontend frameworks and build tools implement hot module reloading. 
When doing so, they fetch the same resource (~same URL), but forcibly bypass the browser's cache.
In Vite's case, [this is done by appending a timestamp to the URL](https://bjornlu.com/blog/hot-module-replacement-is-easy#module-invalidation).

In a browser native module fetcher, we need a way of achieving the same thing.
The chain of module import statements would start with
```ts
import myShader from "../shaders/colors.wgsl?t=42"
```

And continue with
```wgsl
public module foo for "./foo.wgsl?t=42";

...
```



### Supplying modules

The proposal covers the case of a single project, with all modules under the programmer's control.
A single project or library can easily be bundled into a single WGSL string.

However, consuming libraries is more involved. 
The first step is obtaining a library. The library can come from a package manager, a downloaded file or any other source.
Then, we need to supply the library to our project. For this, we want a host language API.

For example, given the following shader code
```ts
const A = lygia::math::PI;
```

The host API would explicitly supply a `lygia` library module. 
```ts
compileShader({
  code: "const A = lygia::math::PI;",
  modules: {
    lygia: ...
  }  
})
```

This API choice allows different ecosystems to be flexible with how they fetch libraries.

TODO: Explain that lygia can appear twice and how this solves it

For any high level API, the supplied modules would also be in the high level API.
For example, in the case of WESL, one would supply the source `.wesl` files to the shader compilation function.
This is done to enable proper IDE support with "go to definition" being usable.

