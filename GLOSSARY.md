# Glossary of WGSL Importing terms
* **WESL** - The extended WGSL language, and is pronounced like "weasel". Stands for WebGPU Enhanced Shading Language
* **WESL translator** - a program that implements the WESL specification
  and transpiles WESL source code to WGSL.
* **WESL linker** - a WESL translator.
  Part of the process of translating WESL involves
  linking together multiple WGSL and WESL modules.
  Also akin to **bundler** in the JavaScript/TypeScript world.
* **Importable item**
  * Structs
  * Functions
  * Type aliases
  * [Const declarations, override declarations](https://www.w3.org/TR/WGSL/#value-decls)
  * [Var declarations](https://www.w3.org/TR/WGSL/#var-decls)
* **Module**: A unit of WESL or WGSL code with its own top-level scope, stored in a single module source.
* **Module Source**: The stored text of a module, typically a WESL or WGSL file.
* **Root Module**: The WESL module from which translation starts. A single project can have many root modules.
* **Declaration Path**: A fully qualified `::`-separated path whose final segment names a declared item.
* **Module Path**: A `::`-separated path naming a module; equivalently, a declaration path minus its final segment. See [Imports](Imports.md#resolving-a-declaration-path).
* **Import Path**: The `::`-separated path written in an import statement. An import collection (`{}`) flattens into separate imports, each with its own import path.
* **Package Module**: The top-level module of a package, typically the file `package.wesl` in the package root. A module path consisting only of `package` or a bare package name refers to the package module.
* **Side effects**: WESL/WGSL shader code that is visible to host code (e.g. in Rust or JavaScript).
  Changes to that shader code have the side effect of changing the host interface to the shader.
  * Things that are specified when [creating a WGSL pipeline](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createRenderPipeline#fragment_object_structure)
    * Shader entry-points
    * Pipeline-overridable constants
    * Global variables, including bindings
  * [Directives](https://www.w3.org/TR/WGSL/#directives): Generated WGSL code must agree on a set of directives
  * (Maybe `const_assert`?)
* **Package**: A publishable body of WESL code containing multiple files. Akin to a JavaScript npm package or a Rust crate.
* **Package Root**: The root directory containing WESL files
* **Visibility**: Which modules can reference or re-export an item. Visibility
  has three levels: `public`, *package* (the default), or `private`. For a
  pipeline-relevant item, visibility also controls whether the root module
  exposes it to the host (see [Visibility](Visibility.md)).
