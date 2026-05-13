# Name mangling
Linker will mangle names internally, but we're hoping to keep the mangling details out of the public api.

See also renaming host visible names in [Visibility](./Visibility.md).

(TBD)

* Extend scheme to support generics.
* Can we find an explicit naming scheme where packages specify names for their host exposed WESL parts
  rather than publishing automatically mangled names?
  * e.g. `@publicName("mypkg_util_bar") @compute fn compute_shader() {}`
  * that way the host code doesn't need to change if the WESL file structure is refactored.

## What gets mangled
All declaration identifiers get mangled, except:

* root module declarations (this is being discussed, see <https://github.com/webgpu-tools/wesl-js/issues/98>)
* declarations imported in the root module
  * if the import renames the declaration (using the `import foo as bar` syntax), the declaration is renamed.
  * if the import is a module (`import some::module` where `some/module.wesl` exists) it does not affect mangling.

## Mangling
To join multiple modules into one, we need to make sure that names do not collide. This is done by name mangling.

Mangling is the process of renaming a declaration and its references to avoid name collisions.
It is not easy: because of name shadowing, local-scope declarations can have the same name as module-scope ones.
To rename all references to a declaration, one has to walk the syntax tree while keeping track of the scope (list of local declarations).

Compiled WGSL files have a few exposed parts, namely the names of the entry functions and pipeline overridable constants (together called "host-visible names").

## Underscore-count mangling
For sharing tests, we need a stable and predictable scheme. This one is also reversible and human-readable.

For name mangling, we introduce the concept of a **fully qualified path**.
This is the absolute path to the module, plus the item. e.g. `bevy_pbr::lighting::main`
1. `_` is the path separator
2. Split into segments (`bevy_pbr`, `lighting`, `main`)
3. If a segment contains `_`, prefix it with `_n` where `n` is the number of underscores it contains. (`_1bevy_pbr`, `lighting`, `main`)
4. The result is joined with `_`. (`_1bevy_pbr_lighting_main`)

## Only-conflict mangling
A simple mangling strategy is

* load everything in a predictable order
* when emitting, keep track of which identifier names have already been used at the global scope
* if one identifier name is used twice, give the second one a name with an incrementing number appended to it

This strategy is very human readable, but cannot be unmangled, since information is lost.
For reproducible builds: It relies on the linker assigning these names in a predictable order. This could be violated by hashmaps, async code or incremental re-compilation, so special care must be taken.
