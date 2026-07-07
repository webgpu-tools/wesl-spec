# Packaging

WESL enables shader packages for reusing shader code by other packages or applications. Shader packages are published to repositories such as [npm] or [crates.io].

## Using shader packages

The WESL linker is responsible for finding and downloading dependencies: [wesl-js] will fetch packages from [npm], while [wesl-rs] will fetch them from [crates.io].
The package dependencies of a shader project are either discovered automatically by the WESL linker, or explicitly listed in the `dependencies` section of [`wesl.toml`].
Inside shader modules, one can reach declarations in dependencies with import paths, as explained in [imports].

## Creating and publishing shader packages

Any [`wesl.toml`] file declares a new shader package, which can be published using the linker's CLI.
Publishing packages is implementation-specific, linkers ([wesl-js] or [wesl-rs]) provide documentation on how to publish to their respective package repositories.

### Package naming guidelines

A package can be published under any name. We however recommend following these guidelines, so the package can be easily found and consumed by end users.

* Use a name that is also a valid WGSL identifier. Otherwise, end-users will have to rename it in [`wesl.toml`].
* Add the `_wgsl` suffix to the name: `mypackage_wgsl`.
* Use snake_case (underscores to separate words): `my_great_package_wgsl`.
* If your package is part of a larger project, or produced by a company, you can prefix it with that name: `mycompany_mypackage_wgsl`.
  * For [wesl-js] specifically, you can use the common naming convention `@mycompany/mypackage_wgsl`.[^1]
* Look for packages with the same name in both [npm] and [crates.io].
  * It is courtesy to leave the name free for the original author if they wish to publish to the other registry.
  * It also avoids confusion for end-users who may think it is the same package.

[^1]: When a package name contains `/` or `@`, [wesl-js] will sanitize it to become a valid WGSL identifier. Refer to its documentation for more information.

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
If two packages depend on semver-compatible versions of `random`, there will be two copies the `rand()` function and its internal state. Calls to `rand()` from both packages refer to the same declaratione.
If however they depend on non-semver-compatible versions, there will be two copies the `rand()` function and its internal state. Calls to `rand()` from both packages refer to distinct declarations.

## Visibility

A package can only import declarations in its direct dependencies, using an import statement or an inline import (see [imports]). 
It cannot import declarations in its indirect dependencies (i.e., dependencies of dependencies).
Declarations in indirect dependencies can be indirectly used by the package, for instance if a direct dependency provides a function which calls a function in one of its dependencies.

Future iterations of WESL may introduce a re-export and/or visibility control mechanism. See [visibility].


[npm]: https://www.npmjs.com/
[crates.io]: https://crates.io/
[`wesl.toml`]: WeslToml.md
[imports]: Imports.md
[visibility]: Visibility.md
[wesl-js]: https://github.com/wgsl-tooling-wg/wesl-js
[wesl-rs]: https://github.com/wgsl-tooling-wg/wesl-rs

