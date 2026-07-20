# Inline Functions

Inline functions are inlined at the calling site, meaning the function body is “copied” in place of the call expression. Inline functions have both advantages and limitations compared to typical functions.

## Syntax

Function declarations preceded with the `@inline` attribute are marked as inline functions.

## Behavior

Inline functions behave somewhat like macros, in the sense that their contents are inlined at the call site, and the declaration is erased at compilation time. 

However, inline functions are hygienic, meaning that they can only access declarations in scope. In-scope declarations are module-scope declarations and function parameters.

(TODO) should we allow discard statements? diagnostics? what should be forbidden in inline functions?

### Parameter and Return Types

Inline functions lift some limitations on allowed parameters and return types. In WGSL, a function return type must be constructible. The return type of an inline function can be a constructible type, a texture, sampler, or pointer type. An inline function may also return a parameter or a module-scope declaration, including bindings. Returning parameters or module-scope declarations does not perform a copy, instead they must are “returned by reference” and can only be bound to a variable.

In WGSL, function-scope var-declarations must have a constructible type. Exceptionally, non-constructible types returned from inline functions can be bound to a var-declaration. 
The scope, address space and access mode of the declaration is inferred from the inline function return value. (TODO: should we allow explicit? in particular the access mode, if applicable.)
Non-constructible types cannot be bound to value declarations.

NOTE: In inline functions returning non-constructible types, the code path leading to the return statement cannot contain branches, so the returned value is determined at shader-creation time. But the function can contain runtime side-effects alongside.

Inline functions can call other inline functions, as well as non-inline functions.

### Limitations

Entry-points functions cannot be inline.

Inline functions can only be invoked from inside a function body.
(TODO: an inline function containing a single return statement could be inlined in const-expressions. Though, this is limited too and could be a job better suited to user-defined `@const` functions)

Inline functions with a return value *must* contain a single return statement at the end of the function body (the last statement), and *must not* contain any early return. This rule is only checked after full resolution of conditional translation attributes.
(TODO can we lift this limitation? we could copy-paste the code following the function invocation in all return branches. This could lead to unexpectedly large code, and unexpected non-uniform control flow. But is much more powerful otherwise.)

NOTE: This ensures that the macro outer control flow has no branch and can be inlined without affecting the following code.

NOTE: Compile-time branching can be performed inside inline functions with the help of conditional translation. See the example below.

## Implementation

### Validation

The implementation *should* check that the returned value from the inline function matches the function return type. Otherwise, the code generation may produce valid WGSL code from invalid WESL code.

The implementation *must* ensure that non-constructible returned values can only be bound to var-declarations. The var-declaration must not contain an explicit address space or access mode. It is inferred. (TODO)

Other cases of invalid use of inline functions would generate invalid WGSL code, therefore validation is optional.

### Code Generation

Implementations are free to generate code differently. This is an example of a conforming code transformation.

1. Rename all identifiers declared in the inline function. See Name Mangling.
2. Insert the inline function statements before the statement containing the call expression, ignoring the return statement.
3. 
  a. If the inline function returns an identifier referring to a module-scope declaration, or a parameter to the inline function, and the call expression is part of a var-declaration: The var-declaration is removed, and all references to the declaration are renamed to the returned identifier.
  b. Otherwise, replace the call expression with the returned expression.

NOTE: If the function is called from a call statement: a simpler code generation is to replace the call statement with a compound statement containing the function body.

### Name Mangling

When renaming identifiers, the implementation is free to pick any name that does not conflict with identifiers in scope at the call site.

We recommend the following renaming scheme for identifiers declared in inline functions:

(TODO)

## Examples

(TODO)

The following snippet:

```
@group(0) @binding(0) sampler_a: sampler;
@group(0) @binding(1) sampler_b: sampler;
@param const USE_SAMPLER_B: bool = false;
// …


@inline fn selectSampler(switch: bool) -> sampler {
    @if(USE_SAMPLER_B) { return sampler_b; }
    @else { return sampler_a; }
}

fn main() {
    // …
    var mySampler = selectSampler();
    textureSample(myTexture, mySampler, myCoords);
}
```

Can be transformed into:
```
@group(0) @binding(0) sampler_a: sampler;
@group(0) @binding(1) sampler_b: sampler;
// …

fn main() {
    // …
    textureSample(myTexture, sampler_a, myCoords);
}
```
