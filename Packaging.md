# Packaging

WESL enables shader packages for reusing shader code by other packages or applications. Shader packages are typically published to repositories such as [npm] or [crates.io].

## Using shader packages

In development, packages are typically loaded from repositories by ecosystem package managers like npm or cargo. [wesl-js] will fetch packages from [npm], while [wesl-rs] will fetch them from [crates.io].
The package dependencies of a shader project are either provided manually using linker-specific settings, discovered automatically by the linker, or explicitly listed in the `dependencies` section of [`wesl.toml`][^1].
A package contents is reachable from user code via module paths starting with the package name. This mechanism is detailed in the [import resolution algorithm](Imports.md#resolving-a-declaration-path).

## Creating and publishing shader packages

Any [`wesl.toml`] file declares a new shader package, which can be published using the linker's CLI.
Publishing packages is implementation-specific, linkers ([wesl-js] or [wesl-rs]) provide documentation on how to publish to their respective package repositories.

### Package naming guidelines

A package can be published under any name. We however recommend following these guidelines, so the package can be easily found and consumed by end users.

* Use a name that is also a valid WGSL identifier. Otherwise, end-users will have to rename it in [`wesl.toml`].
* Add the `_wgsl` suffix to the name: `mypackage_wgsl`.
* Use snake_case (underscores to separate words): `my_great_package_wgsl`.
* If your package is part of a larger project, or produced by a company, you can prefix it with that name: `mycompany_mypackage_wgsl`.
  * For [wesl-js] specifically, you can use the common naming convention `@mycompany/mypackage_wgsl`.[^2]
* Look for packages with the same name in both [npm] and [crates.io].
  * It is courtesy to leave the name free for the original author if they wish to publish to the other registry.
  * It also avoids confusion for end-users who may think it is the same package.
  * 
## Semver-compatibility and dependency unification

[wesl-js] and [wesl-rs] both delegate package installation and version management to npm or Cargo. 
These package managers typically apply  _dependency version unification_: if two packages in the dependency tree are [semver-compatible](https://semver.org/), the package manager installs only one version of the package, often the latest semver-compatible version available. 
For WESL, dependency unification can have observable side-effects; for instance, module-scope declarations may or may not be duplicated.

### Example

```wgsl
// package random
// --------------
var<private> prng_state: f32 = 0;

// return a random float in [0, 1]. Do not actually use this function, it is awful.
fn rand() -> f32 {
    prng_state = fract(sin(x)*12345.6789);
    return prng_state;
}
```

In this example, the `random::rand()` function has internal state represented by `prng_state`.
If two packages depend on semver-compatible versions of `random`, there will be a single copy of the `rand()` function and its internal state. Calls to `rand()` from both packages refer to the same declaration.
If however they depend on non-semver-compatible versions, there will be two copies of the `rand()` function and its internal state. Calls to `rand()` from both packages refer to distinct declarations.

## Package visibility

A package can only import from packages that are its direct dependencies, using an import statement or an inline import (see [imports]). 
It cannot import from indirect dependencies (i.e., dependencies of dependencies).
Declarations in indirect dependencies can be indirectly used by the package, for instance if a direct dependency provides a function which calls a function in one of its dependencies.


[npm]: https://www.npmjs.com/
[crates.io]: https://crates.io/
[`wesl.toml`]: WeslToml.md
[imports]: Imports.md
[visibility]: Visibility.md
[wesl-js]: https://github.com/wgsl-tooling-wg/wesl-js
[wesl-rs]: https://github.com/wgsl-tooling-wg/wesl-rs

[^1]: At the time of writing, [wesl-js] and [wesl-rs] do not support reading dependencies in `wesl.toml`, but that will become the preferred way of discovering dependencies in the near future.
[^2]: When a package name contains symbols such as `-` `/` or `@`, [wesl-js] will sanitize it to become a valid WGSL identifier, e.g. a package named `@mycompany/mypackage_wgsl` can be imported with the prefix `mycompany__mypackage_wgsl`. [wesl-js] forbids symbols, with the exception of `-` (dash) which is replaced with `_` (underscore). Refer to the linkers' documentation for more information.

