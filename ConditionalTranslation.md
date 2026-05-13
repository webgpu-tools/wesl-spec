# Conditional Translation

## Overview
> *This section is non-normative*

Conditional translation is a mechanism to modify the output source code based on parameters passed to the *WESL translator*.
This specification extends the [*attribute* syntax](https://www.w3.org/TR/WGSL/#attributes) with a new `@if` attribute.
This attribute indicates that the syntax node it decorates can be removed by the *WESL translator* based on feature flags.

> *This implementation is similar to the `#[cfg(feature = "")]` syntax in Rust.*

### Usage Example

```wgsl
// global variables and bindings...
@if(textured)
@group(0) @binding(0) var my_texture: texture_2d<f32>;

// structs declarations and struct members...
struct Ray {
  position: vec4f,
  direction: vec4f,
  @if(debug_mode && raytracing_enabled)
  ray_steps: u32,
}

// function declarations, parameters and statements...
fn main() -> vec4f {
  @if(legacy_implementation || (is_web_version && xyz_not_supported))
  let result = legacy_impl();
  @if(!legacy_implementation && !(is_web_version && xyz_not_supported))
  let result = modern_impl();
}
```

Quirky examples

```wgsl
// attribute order does not matter.
@compute @if(feature) fn main() { }
// ... is equivalent to 
@if(feature) @compute fn main() { }

// feature names live in their own namespace, i.e. they cannot shadow, or be shadowed by declarations.
const feature1 = 10;
@if(feature1) fn main() -> u32 { // 'feature1' in @if does not refer to the const-declaration.
    return feature1*2;           // 'feature1' in return statement does not refer to the feature flag.
}
```

## Definitions
* **Translate-time expression**: A *translate-time expression* is evaluated by the *WESL translator* and eliminated after translation.
  Its grammar is a subset of normal WGSL [expressions](https://www.w3.org/TR/WGSL/#expressions). It must be one of:
  * a *translate-time feature*,
  * a [logical expression](https://www.w3.org/TR/WGSL/#logical-expr): logical not (`!`), short-circuiting AND (`&&`), short-circuiting OR (`||`),
  * a [parenthesized expression](https://www.w3.org/TR/WGSL/#parenthesized-expressions),
  * a boolean literal value (`true` or `false`).

* **Translate-time scope**: The *translate-time scope* is an independent scope from the [*module scope*](https://www.w3.org/TR/WGSL/#module-scope), meaning it cannot see any declarations from the source code, and its identifiers are independent.

* **Translate-time feature**: A *translate-time feature* is an identifier that evaluates to a boolean. It is set to `true` if the feature is *enabled* during the translation phase and `false` if the feature is *disabled*. It lives in the *Translate-time scope*.

* **Translate-time attribute**: A *translate-time attribute* is parametrized by a *translate-time expression*. It is eliminated after translation but can affect the syntax node it decorates.

## Location of *Translate-time attributes*
A *translate-time attribute* can appear before the following syntax nodes:

* [directives](https://www.w3.org/TR/WGSL/#directives)
* [variable and value declarations](https://www.w3.org/TR/WGSL/#var-and-value)
* [type alias declarations](https://www.w3.org/TR/WGSL/#type-alias)
* [const assertions](https://www.w3.org/TR/WGSL/#const-assert-statement)
* [function declarations](https://www.w3.org/TR/WGSL/#function-declaration-sec)
* function formal parameter declarations
* [structure declarations](https://www.w3.org/TR/WGSL/#struct-types)
* structure member declarations
* [statements](https://www.w3.org/TR/WGSL/#statements)
* [switch clauses](https://www.w3.org/TR/WGSL/#switch-statement)

> [!TIP]
> *Translate-time attributes* are not allowed in places where removal of the syntax node would lead to syntactically incorrect code. The current set of *translate-time attribute* locations guarantees that the code is syntactically correct after specialization. This is why *translate-time attributes* are not allowed before expressions.

### Update to the WGSL grammar
The WGSL grammar allows attributes in several locations where *translate-time attributes* are not allowed (1). Conversely, the WGSL grammar does not allow attributes in several locations where *translate-time attributes* are allowed (2).

Refer to the [updated grammar appendix](#appendix-updated-grammar) for the list of updated grammar non-terminals.

1. A *translate-time attribute* CANNOT decorate the following syntax nodes, even if the WGSL grammar allows attributes before these syntax nodes:
   * function return types
   * the body (part surrounded by curly braces) of:
     * function declarations
     * switch statements
     * switch clauses
     * loop statements
     * for statements
     * while statements
     * if/else statements
     * continuing statements

2. The grammar is extended to allow *translate-time attributes* before the following syntax nodes:
   * const value declarations
   * variable declarations
   * directives
   * struct declarations
   * switch clauses
   * assignment statements
   * increment and decrement statements
   * break statements
   * break-if statements
   * continue statements
   * continuing statements
   * return statements
   * discard statements
   * function call statements
   * const assertion statements

> [!WARNING]
> We added attributes on assignment and increment/decrement statements. This change makes the WESL grammar no longer [LR(1)](https://en.wikipedia.org/wiki/LR_parser). See [`wesl-rs#162`](https://github.com/webgpu-tools/wesl-rs/issues/162) for details.
>
> Due to this limitation, *wesl-rs* does not allow attributes on assignment, increment and decrement statements starting with a `(`. Concretely, this is not supported by *wesl-rs*: `@if(FOO) (x)++`.

## `@if` attribute
The `@if` *translate-time attribute* is introduced. It takes a single parameter. It marks the decorated node for removal if the parameter evaluates to `false`.

A syntax node may at most have a single `@if` attribute. This keeps the way open for a `@else` attribute introduction in the future.
Checking for multiple features is done with an `&&`

```wgsl
@if(feature1 && feature2)   const decl: u32 = 0;
```

> *See the [possible future extensions](#possible-future-extensions) for the attributes `@elif` and `@else`.
> They may be introduced in the specification in a future version if deemed useful.*

Example:

```wgsl
@if(feature_1 && (!feature_2 || feature_3))
fn f() { ... }
@if(!feature_1)                               // corresponds to @elif(!feature_1)
fn f() { ... }
@if(feature_1 && !(!feature_2 || feature_3))  // corresponds to @else
fn f() { ... }
```

## Execution of the conditional translation phase
1. The *WESL translator* is invoked with the list of features to *enable* or *disable*.

2. The source file is parsed.

3. The *translate-time features* in *translate-time expressions* are resolved:
   * If the feature is *enabled*, the identifier is replaced with `true`.
   * If the feature is *disabled*, the identifier is replaced with `false`.

4. *Translate-time attributes* are evaluated:
   * If the decorated syntax node is marked for removal: it is eliminated from the source code along with the attribute.
   * Otherwise, only the attribute is eliminated from the source code.

5. The updated source code is passed to the next translation phase. (e.g. import resolution)

### Incremental translation
In case some features can only be resolved at runtime, a *WESL translator* can *optionally* support feature specialization in multiple passes:

* In the initial passes, the *WESL translator* is invoked with some of the feature flags. It replaces their occurrences in *translate-time attributes* with either `true` or `false`.
  These passes return a partially-translated WESL code.
* After the final pass, the resulting code must be valid WGSL. It is a *link-time error* if any used *translate-time feature* was not provided to the linker.

If the *WESL translator* does not support incremental translation, it is a *link-time error* if any used *translate-time feature* was not provided to the linker.

> *It is not an error to provide unused feature flags to the linker. However, an implementation may choose to display a warning in that case.*

## Possible future extensions
> *This section is non-normative*

* `@else` and `@elif` attributes:
  * An `@elif` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It takes one parameter.
     It marks the decorated node for removal if its parameter evaluates to `false` OR if any of the previous `@if` and `@elif` attribute parameters evaluate to `true`.
  * An `@else` attribute decorates the next sibling of a syntax node decorated by a `@if` or an `@elif`. It does not take any parameter.
     It marks the decorated node for removal if any of the previous `@if` and `@elif` attribute parameters evaluate to `true`.

  The `@else` attribute has the nice property that all cases lead to generating a node, and *could* therefore be used in places where the node is required (e.g. expressions)

  Example:

  ```wgsl
  @if(feature_1 && (!feature_2 || feature_3))
  fn f() { ... }
  @elif(!feature_1)
  fn f() { ... }
  @else
  fn f() { ... }
  ```

* High-complexity *translate-time expressions*: if we end up implementing other *translate-time attributes*, such as loops (e.g. `@for`, `@repeat`), or [translate-time-evaluable](https://zig.guide/language-basics/comptime/) expressions, then we would need to extend the grammar of *translate-time expressions*. It would also affect this proposal.

  *Example*

  ```wgsl
  @comptime
  fn response_to_the_ultimate_question() -> u32 {
    return 42;
  }
  
  @if(response_to_the_ultimate_question() == 42)
  fn f() { ... }
  ```

* Decorating other WESL language extensions: import statements could be decorated with *translate-time attributes* too.

  *Example*

  ```wgsl
  @if(use_bvh)
  import accel/bvh_acceleration_structure as scene_struct;
  @else
  import accel/default_acceleration_structure as scene_struct;
  ```

## Appendix: Updated grammar
The following non-terminals are added or modified:

```grammar
    diagnostic_directive :
      attribute * 'diagnostic' diagnostic_control ';'

    enable_directive :
      attribute * 'enable' enable_extension_list ';'

    requires_directive :
      attribute * 'requires' software_extension_list ';'

    struct_decl :
      attribute * 'struct' ident struct_body_decl
     
    type_alias_decl :
      attribute * 'alias' ident '=' type_specifier

    variable_or_value_statement :
      variable_decl
    | variable_decl '=' expression
    | attribute * 'let' optionally_typed_ident '=' expression
    | attribute * 'const' optionally_typed_ident '=' expression

    variable_decl :
      attribute * 'var' _disambiguate_template template_list ? optionally_typed_ident
     
    global_variable_decl :
      variable_decl ( '=' expression ) ?

    global_value_decl :
      attribute * 'const' optionally_typed_ident '=' expression
    | attribute * 'override' optionally_typed_ident ( '=' expression ) ?

    case_clause :
      attribute * 'case' case_selectors ':' ? compound_statement

    default_alone_clause :
      attribute * 'default' ':' ? compound_statement

    assignment_statement :
      attribute * lhs_expression ( '=' | compound_assignment_operator ) expression
    | attribute * '_' '=' expression

    increment_statement :
      attribute * lhs_expression '++'

    decrement_statement :
      attribute * lhs_expression '--'

    break_statement :
      attribute * 'break'

    break_if_statement :
      attribute * 'break' 'if' expression ';'

    continue_statement :
      attribute * 'continue'
     
    continuing_statement :
      attribute * 'continuing' continuing_compound_statement

    return_statement :
      attribute * 'return' expression ?
    
    discard_statement:
      attribute * 'discard'

    func_call_statement :
      attribute * call_phrase

    const_assert_statement :
      attribute * 'const_assert' expression

    statement :
      ';'
    | return_statement ';'
    | if_statement
    | switch_statement
    | loop_statement
    | for_statement
    | while_statement
    | func_call_statement ';'
    | variable_or_value_statement ';'
    | break_statement ';'
    | continue_statement ';'
    | discard_statement ';'
    | variable_updating_statement ';'
    | compound_statement
    | const_assert_statement ';'

    attribute :
      '@' ident_pattern_token argument_expression_list ?
    | align_attr
    | binding_attr
    | blend_src_attr
    | builtin_attr
    | const_attr
    | diagnostic_attr
    | group_attr
    | id_attr
    | interpolate_attr
    | invariant_attr
    | location_attr
    | must_use_attr
    | size_attr
    | workgroup_size_attr
    | vertex_attr
    | fragment_attr
    | compute_attr
    | if_attr

    if_attr:
      '@' 'if' '(' expression ',' ? ')'
```
