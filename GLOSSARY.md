# Glossary of WGSL Importing terms

## General

* **WESL**: The extended WGSL language, and is pronounced like "weasel". Stands for WebGPU Enhanced Shading Language.
* **WESL linker**: A program that implements the WESL specification and transpiles WESL source code to WGSL.
  Part of the process of translating WESL involves linking together multiple WGSL and WESL modules.
  Also akin to **bundler** in the JavaScript/TypeScript world.
* **Mangling**: link process in which certain declarations are renamed to avoid name conflicts.
* **Side effects**: WESL/WGSL shader code that is visible to host code (e.g. in Rust or JavaScript).
  Changes to that shader code have the side effect of changing the host interface to the shader.
  * Things that are specified when [creating a WGSL pipeline](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createRenderPipeline#fragment_object_structure)
    * Shader entry-points
    * Pipeline-overridable constants
    * Global variables, including bindings
  * [Directives](https://www.w3.org/TR/WGSL/#directives): Generated WGSL code must agree on a set of directives
  * (Maybe `const_assert`?)
* **Visibility**: An item's visibility defines which modules can reference or re-export it.
  Visibility has three levels: `public`, *package* (the default), or `private`.
  For a pipeline-relevant item, visibility also controls whether the entry module exposes it to the host. see [Visibility](Visibility.md).

## Modules

* **Module**: A unit of WESL or WGSL code with its own top-level scope, stored in a single module source.
* **Module Source**: The stored text of a module, typically in a WESL or WGSL file.
* **Entry Module**: The WESL module from which translation starts. Its public declarations form the **shader-host interface** and are not mangled. A single application can have many entry modules.
* **Module Path**: A `::`-separated path naming a module; equivalently, a declaration path minus its final segment.
* **Declaration Path**: A `::`-separated path whose final segment names a declared item.
* **Canonical Path**: A fully qualified module/ declaration path (which does not contain any `super::`). There is exactly one canonical path per module or declaration within a package.
* **Importable item**: A module declaration or re-export that can be imported by other modules.
  * Structs
  * Functions
  * Type aliases
  * [Module-scope `const`, `override` and `var` declarations](https://www.w3.org/TR/WGSL/#var-and-value)
  * Re-exports (aka. public imports)

## Packages

* **Package**: A publishable body of WESL code containing multiple files. Akin to a JavaScript npm package or a Rust crate.
* **wesl.toml**: The optional configuration file for a package. See [WeslToml](WeslToml.md).
* **Module Path Resolution**: Mapping between module paths and module source within a package. Choice of mapping is implementation-specific.
* **Filesystem Resolution**: The default module path resolution. It maps module paths to file paths relative to a **Package Root Directory**. See [Filesystem Resolution][Imports.md#filesystem-resolution].
* **Package Root Module Path**: The module path consisting only of `package`. Corresponds to `package.wesl` in the package root directory with the filesystem mapping.
