# Do Blocks (experimental)

Do blocks let you schedule compute passes and fragment shader draws on the GPU
from a WESL module in [wgsl-play](README.md) and [wgsl-test](../wgsl-test).
You can choose buffers, run compute passes in loops or recursive calls, and
set values for each pass. It's all in WESL, no JavaScript required.

A do block is basically a WGSL function that runs on the CPU. Do blocks use
the same statements, expressions, and types as WGSL/WESL, plus a few
host-side additions such as `@dispatch`. Similarly, a `do var` is a
module-scope variable on the CPU side. Do code shares constants, struct
types, and buffers with the shader code around it, and calls shader entry
points to produce GPU commands that are set up and dispatched to the GPU to
run. [Do-code language](#do-code-language) lists the differences from WGSL.

[wgsl-play](README.md) runs the program and displays its output.
[wgsl-test](../wgsl-test) runs do blocks as GPU tests.
[wgsl-edit](../wgsl-edit) provides syntax highlighting and parser and type
diagnostics. Other WESL tools don't yet support `wesl_do_blocks`.

## Contents

- [Enable do blocks](#enable-do-blocks)
- [Dispatch compute shaders](#dispatch-compute-shaders)
- [Add parameters to entry points](#add-parameters-to-entry-points)
  - [Buffer parameters](#buffer-parameters)
  - [Value parameters](#value-parameters)
  - [Binding struct parameters](#binding-struct-parameters)
  - [Builtin parameters](#builtin-parameters)
  - [Pass arguments](#pass-arguments)
- [Draw with a fragment shader](#draw-with-a-fragment-shader)
- [Work with buffers](#work-with-buffers)
  - [Persistent buffers](#persistent-buffers)
  - [Allocate buffers with `createBuffer<T>(n)`](#allocate-buffers-with-createbuffertn)
- [Preserve CPU state with `do var`](#preserve-cpu-state-with-do-var)
- [Call other do blocks](#call-other-do-blocks)
  - [Control the main block from the UI](#control-the-main-block-from-the-ui)
- [Use multiple modules](#use-multiple-modules)
- [Test do blocks](#test-do-blocks)
- [Do-code language](#do-code-language)
- [Examples](#examples)

## Enable do blocks

Add `requires wesl_do_blocks;`. Here's an example you can try on
[wgsl-play.dev](https://wgsl-play.dev/#v1=N4IgDgTg9gVgpgYwC4gFygO5wM4BsDKECaoYAhggNZkDmcqqAtmQJYB2aIEcAjgK4tu2AARY8AfQAmUcQCNcUKtgDcAHTbrpwgG5kIwyWSRlhAXmEJuRuACE+AM3twIAHj4BmAEwA+ABReASmVhEOEAejDhMlwFBGso4WwkKAhaOGFZByd9dmEAdQBRfAAZYXV1AAEEKEYwPiR0iowUyhpoPjBxbBYALzhfABYA9Xs2YXsWGN9JJNRhMCRXJJS0gBooiFSATzcvb3WrSXEMCBYG-ZCI4Tg2Ra35qHYkeb0yRjgGiGx1UN+daL49GEHk8qx+fxCFUykyQ7F8NAUsmi4nY2kURhYUDYKMkAWELEkc20iHcfDxwHBMyQAG0CQA6AAeAF1hABqcy6XCAtRsAC+5TYWggfDYvnJ4IqkhY2HISAQAAtfJ4yeNJrhpkYyOtlUEIaErlKZUYFQlqrV6ulsPKyJJnBLDbKFYMVRMpoZjOsAIwABjJPP5HF5qy4UCgSAAslBJHxcHAAHJvOCcZjsEDB6qCs6YtjYEi8oMgWFIWOcG5keRwAC00kr8kUlFzvKAA):

```wgsl
requires wesl_do_blocks;

do var data = createBuffer<u32>(32);    // allocate a storage buffer in WESL 

@compute @workgroup_size(4)
fn fill(dst: ptr<storage, array<u32>, read_write>,  // entry point parameters
        value: u32,
        @builtin(global_invocation_id) id: vec3u) {
  dst[id.x] += value;
}

do run() {
  @dispatch(2u) fill(data, 2u);         // dispatch a compute shader
  @dispatch(4u) fill(data, 10u);
}
```

The host runs `do run()` once. The dispatches run in order. In this case,
the first dispatch launches 2 workgroups of 4 invocations and so modifies
the first 8 elements in the buffer. The second dispatch launches 4
workgroups, modifying 16 elements. So the first 8 elements in the array are
12, the next 8 elements are 10, and the final 16 elements remain untouched.


## Dispatch compute shaders

`@dispatch(...)` takes 1 to 3 workgroup counts, setting how many workgroups
of that entry point run on the GPU. The do blocks host library converts
`@dispatch` statements into WebGPU
[`dispatchWorkgroups`](https://developer.mozilla.org/en-US/docs/Web/API/GPUComputePassEncoder/dispatchWorkgroups)
calls.

## Add parameters to entry points

In a module with do blocks, compute and fragment entry points can declare
ordinary parameters next to their `@builtin` ones. Do code supplies the
arguments on each call. This example passes a buffer, a struct of values,
and a scalar.

[Example on wgsl-play.dev](https://wgsl-play.dev/#v1=N4IgDgTg9gVgpgYwC4gFygO5wM4BsDKECaoYAhggNZkDmcqqAtmQJYB2aIEcAjgK4tu2AARY8AfQAmUcQCNcUKtgA0onLilwAZnAjdJc9pPY1sAbgA6bK9iQQ+yYfigQkABTIQyjEcGHYACxYtJFRhPgBmACZVcmxsABk4NjDIqOEAXysAN08AHlsXWjhVbjIDDAgWJDgAPn8gkOwwzy8ATzy01QAWWstrNmlhezYACgBKYWArYWEtF2FR3IhhOOwAOT5GYQBeYQAGPjNVsnjN7bzhbqOTs62AanvJ6bZZ2dw4JBOvH12nF3cnm82FGa3OwgAVFc+D0+ON+m9hAABYzYchIBABUYARjhw0QLkk+EaSFGADJAsEkCpvsDYqcNlt4cIAPQs24iTw0LbJakzTJWLIDJEIKCMMB8GrIjAuSg0aB8MDibAsABecBx4ysWle3FFECJJNGklsYTAdgKSCKdFUrTIHTStVKcHK4kq1Tqyn5iJ9iPIP2a-1cHgD9PiAElBnAAB6paJe16+pPI2QCXBIdijGgKWRkDTsbKKMgZqBscQsSSTCthbKICJ4l6zYIcyOSGPCS7+4EAOjWSVejbeJqQAG01q2YwBdP5dnzdykhBECthCkAZZRcKBQJAAWSgkj4H3W3jgnGY7BAG9Fg2qLFL2BIGXXIAzSA+nDWAFpZ59dA+MkAA):

```wgsl
requires wesl_do_blocks, wesl_deferred_bindings;

struct SortParams { shift: u32, passLen: u32 }
var<storage, read_write> shifts: array<u32, 4>;

do run() {
  for (var passNum = 0u; passNum < 4u; passNum++) {
    let params = SortParams(passNum * 4u, 4u);
    @dispatch(1u) recordShift(&shifts, params, passNum); // pass arguments
  }
}

@compute @workgroup_size(1)
fn recordShift(dst: ptr<storage, array<u32>, read_write>,
               params: SortParams, passIndex: u32,
               @builtin(global_invocation_id) id: vec3u) {
  if passIndex < params.passLen {
    dst[passIndex] = params.shift;
  }
}
```

The results panel shows `0, 4, 8, 12`.

### Buffer parameters

To take a storage buffer as an entry point parameter, use a pointer,
`ptr<storage, array<E>, access>`. Use a runtime-sized array. 
Inside the entry point function, use the pointer as in WGSL. 
`arrayLength(dst)` returns the length of the buffer.

When passing a storage buffer pointer, a `read_write`
buffer can go to a `read` parameter, and a fixed-size buffer can go to a
runtime-sized parameter.

You can pass pointers from the entry point shader function to other
shader functions if the browser supports 
the WGSL `unrestricted_pointer_parameters` language feature. 
(Chrome, Safari, and Edge now. Firefox is forthcoming.)

### Value parameters

Value parameters are currently limited to `u32`, `i32`, or `f32` scalars, 
or a struct whose members are scalars. 
You can reuse a struct value, like `params` above, in several dispatches. 

### Binding struct parameters

A binding struct groups buffer pointers into one entry point parameter. 
See [Binding structs](DeferredBindings.md#binding-structs) for details.

### Builtin parameters

Entry points can still take `@builtin` parameters, which the pipeline
supplies as usual. Do blocks don't change them. 
By convention `@builtin` parameters come last in the parameter list.

### Pass arguments

Do code passes arguments to entry points with dispatch syntax: 
`@dispatch(1u) recordShift(...)` for a compute entry point, 
or a plain call for a fragment entry point (see
[Draw with a fragment shader](#draw-with-a-fragment-shader)). The
caller supplies every parameter except the `@builtin` ones.

## Draw with a fragment shader

In wgsl-play, call a fragment entry point without `@dispatch` to draw to
the canvas. Fragment entry points take
[parameters](#add-parameters-to-entry-points) like compute entry points.
wgsl-play supplies a vertex stage that covers the canvas, so the fragment
shader's only pipeline input is `@builtin(position)`.

This module advances 64 counters each frame and draws them as a slowly
scrolling wave. The do vars preserve the two buffer roles between frames.

[Example on wgsl-play.dev](https://wgsl-play.dev/#v1=N4IgDgTg9gVgpgYwC4gFygO5wM4BsDKECaoYAhggNZkDmcqqAtmQJYB2aIEcAjgK4tu2AARY8AfQAmUcQCNcUKtgA0onLilwAZnAjdJc9pPY1sAbgA6bKwDcyEADzYkUCLTiruZAxggskcAB8wjR+kgCCqML2bgCeDnwAzABMqgBsACyBlmx2js6u7p5w3uK+-kEhYQBCUTFk8UmpwpnZVtLCecIIfHpwbEjCALzCAGShLBE5HV1scAAegyPjNTntUMIQfGwAFACUwsBWwsIAAsbY5EgIABY7AIx8B952bAhwOz19A6pzi3s5E64OCDJCMMDDbq9bgDMxQ75LYR-JBw5GQsFgOEAeixwmwGDIENkfC0OggmygwOwxzxNygGE+0P6SABVgAvlYrKcEFBwXwAmcMK5KKEoHwwOJsCwAF4fTJ7KxaNjRSSvd47bBEKJgJD5FxuOiqeqNFKBQLKGknK3WyTObW6pz6orRPQNBKm4qlcoBc2W63W07Eli4JDsHY0BSyMgadg2RRkUNQNjiSYHSZRGyIRJPQ4021IADakwAdPMALqQzUIIuSUsVgDUwkeOQ51jYpy0BsYzMVyuwdIZ71wuGw9r1hUNLri7uSZotyv9J0DAhDYbAUClibYB3Xo86iAyWgOAFpgqcFAgEywkzsAAwHTMIQ+5hfAwYsSE7Jo7Xelg64gAOHMAFIWgyPhsUA48wBYeY4FwYQozYW0aTfWlvDgSFb2LABWYRG2wvCACo8TDLQUk+eCRyLBthA-EjwIOEjsNvRJWQXbgkF6ZVH0PDUbgw1R+0E9DJA8Jti3vFsrBANllC4KAoCQABZKBJD4YEADkyG7ThmHYEB5J5ZD-GvNhsBINk5JAUMkGBThpGPeRFEobBj0kNwMFkoA):

```wgsl
requires wesl_do_blocks, wesl_deferred_bindings;

var<storage, read_write> gridA: array<u32, 64>;
var<storage, read_write> gridB: array<u32, 64>;
do var current = &gridA;
do var next = &gridB;

do run() {
  @dispatch(1u) advance(current, next);
  let tmp = current; current = next; next = tmp; // swap buffer roles
  show(current);
}

@compute @workgroup_size(64)
fn advance(src: ptr<storage, array<u32>>,
           dst: ptr<storage, array<u32>, read_write>,
           @builtin(global_invocation_id) id: vec3u) {
  dst[id.x] = src[id.x] + 1u;
}

@fragment
fn show(cells: ptr<storage, array<u32>>,
        @builtin(position) pos: vec4f) -> @location(0) vec4f {
  let i = (u32(pos.x) / 8u) % 64u; // 8-pixel bands
  let shade = 0.5 + 0.5 * sin(f32(cells[i] + i * 4u) * 0.03);
  return vec4f(shade, shade, shade, 1.0);
}
```

In wgsl-play, a [main block](#control-the-main-block-from-the-ui) that calls a fragment
entry point, directly or through another do block, runs once per frame. A
main block w/o a fragment shader runs once and shows its buffers in the
results panel: the module-scope storage vars, then the main module's
`do var`s that hold a `createBuffer<T>(n)` result.

## Work with buffers

Do code can pass and store pointers to whole buffers. 

### Allocate buffers with `createBuffer<T>(n)`

To create a storage buffer with a size chosen in do code, use
`createBuffer<T>(n)`. It returns a `ptr<storage, array<T>, read_write>` to
a new buffer of `n` elements. 
Buffer contents persist across multiple dispatches in the same run.
Buffer contents are unspecified at creation.

Here's an [example on wgsl-play.dev](https://wgsl-play.dev/#v1=N4IgDgTg9gVgpgYwC4gFygO5wM4BsDKECaoYAhggNZkDmcqqAtmQJYB2aIEcAjgK4tu2AARY8AfQAmUcQCNcUKtgDcAHTbrpwgG5kIwoX1xJhAXmEJuZJHABCfAGYO4EADx8AzACYAfAAoARj4ASmVhcPCAekjhMBdsFmwkEQpobBEIPjZsdU0oAyy-YOFgYQcWXFwAQTZJfD5GQK8ADhCwgF9c2vzyypq6hr8EKCykVGFPL2LgdXDcOBNsS2sEAAszCysbeycXd29-YdHQ2eEAAUlE8iQ1vyGRthMAamEANg8Q4RjXgBZP3twfiWEBWqxObHCFyuoMCn2wg2BoIANAYcEYkKEIl8YrIoEh1no4LEoOwbBBsCi2PkAGTqToaNhnYaMMB8GznDBQCCUGjQPhgcQJABecD8v2C6gcEIBfkkSXGYCQbiSXNocBRehBAE99r4UVZJOIMBAWDYfEjTljIbIBMZ2H4aApZGRcOJ2NpFNYWFA2G7JMUWJJxtpEB9pqcWA5hIGAHQAD2ErmEmrIWoAMnA2DR8bKktNhHKkABtWNxgC6G1LHTpXSZUBZbKJZ053N5IwFwtFAQlbClwnhjWwRAVStcKpBdA1EG1up85stWMLI+VSFVk+T09Ts-1cDIhuNprgPnDEJ0emEq6QLo2AAY+GpTw4ucI-Lp9Cxb-fo4mN9qM1mcyHBBMRYJ4nnzS9ryecwgJLMtq1PQsixvCtzEg3AH3pEB2iRLgoDxABZKBJCMOAADkyEYOBOGYdgQFw4ZalNb1shIdocJAJBTXmThLF3GwAFobV2CBsKAA):

```wgsl
requires wesl_do_blocks;

do var result = createBuffer<u32>(1u);     // persists across runs

do run() { fillAndSum(128u); }

do fillAndSum(count: u32) {
  let scratch = createBuffer<u32>(count);
  @dispatch((count + 63u) / 64u) fill(scratch);
  @dispatch(1u) sum(scratch, result);    // both are pointers, no &
}

@compute @workgroup_size(64)
fn fill(dst: ptr<storage, array<u32>, read_write>,
        @builtin(global_invocation_id) id: vec3u) {
  if id.x < arrayLength(dst) { dst[id.x] = id.x; }
}

@compute @workgroup_size(1)
fn sum(src: ptr<storage, array<u32>>,
       dst: ptr<storage, array<u32>, read_write>) {
  var total = 0u;
  for (var i = 0u; i < arrayLength(src); i++) { total += src[i]; }
  dst[0] = total;
}
```

## Preserve CPU state with `do var`

Use a module-scope `do var` to keep state on the CPU,
such as the buffer pointers in the draw example or a counter declared as
`do var frame = 0u;`. Do vars run their initializers on the first run.

do var types are inferred as in WGSL.
For example, `do var time = 0;` holds an `i32`, so write
`do var time: f32 = 0;` to accumulate `u.delta_time`. Unlike WGSL, a
do-code `var` can hold a pointer.

Swapping `current` and `next` in the draw example changes the buffer each
pointer refers to. It does not copy buffer contents. To read do vars from
JavaScript, use wgsl-play's [`readVars()`](README.md#reading-shader-state).

In wgsl-play, rewind, changing a `@rewind` control, and
[`replayTo()`](README.md#replaying-to-a-time) reset the program.
Module-scope storage buffers are reset to zero, and do vars run their initializers
again on the next run. 

## Call other do blocks

Define a module-scope block as `do name(params) { ... }` and call it as
`fillAndSum(128u)` in the buffer example. Parameters are immutable. Each
call has its own locals. Calls can be recursive.

### Control the main block from the UI

wgsl-play runs the main do block: the one block in the main module that no
other do block calls, not counting blocks marked `@test`. Since nothing calls
the main do block, wgsl-play supplies its arguments from UI controls. You can
annotate scalar parameters of the main do block with `@range` or `@toggle`,
for example `do run(@range(0, 8, 1, 3) passes: u32) { ... }`. A parameter
without an annotation receives 0.

## Use multiple modules

Import do blocks, do vars, and entry points as ordinary WESL declarations,
then call or dispatch them from your main block. Entry point buffer and
value parameters work across modules.

Libraries should take needed resources as ptr parameters, and construct
internal buffers with `createBuffer()`: 

```wgsl
const workgroupSize = 64u;

/// sum first len elements of src into dst[0]
public do blockSum(src: ptr<storage, array<u32>>,
                   dst: ptr<storage, array<u32>, read_write>, 
                   len: u32) {
  let numBlocks = (len + workgroupSize - 1) / workgroupSize;
  let partials = createBuffer<u32>(numBlocks);   // library private buffer
  @dispatch(numBlocks) sumBlocks(src, partials, len);
  @dispatch(1u) sumPartials(partials, dst, numBlocks);
}
```

## Test do blocks

wgsl-test runs each `@test do` block. Dispatch an entry point that calls
wgsl-test assertions to check the results:

```wgsl
import wgsl_test::expectEq;
requires wesl_do_blocks, wesl_deferred_bindings;

var<storage, read_write> data: array<u32, 4>;

@compute @workgroup_size(4)
fn fill(@builtin(global_invocation_id) id: vec3u) {
  data[id.x] = id.x * 10u;
}

@compute @workgroup_size(1)
fn check() { expectEq(data[3], 30u); }

@test do fillThenCheck() {
  @dispatch(1u) fill();
  @dispatch(1u) check();
}
```

A test passes if every run completes with no assertion failure. See
[wgsl-test](../wgsl-test) for `@runs(n)` and for running do blocks from
TypeScript.

wgsl-play.dev support for `@test` is planned.

## Do-code language

A do block is like a WGSL `fn` that runs on the CPU. It uses WGSL
statements: `let`, `var`, and `const` declarations, assignment (including
compound assignment, `++`, and `--`), `if` / `else`, `for`, `while`, and
`loop`, with `break`, `continue`, and `return`. Expressions
use WGSL's operators, conversions such as `u32(x)`, vector, matrix, and
struct constructors, swizzles and member reads, and value builtins such as
`min`, `clamp`, `mix`, and `dot`. Values follow WGSL's type rules. 

Do code integrates with other WGSL code. Do blocks can read the same
consts, struct types, and can call WGSL entry points.
So one WESL program describes both the GPU work and its orchestration.

Do code differs from WGSL in some ways:

- Extensions like `createBuffer()` and `@dispatch` only work from 
  do blocks and do var initializers.
- Do block calls can be recursive.
- A `var` or `do var` can hold a pointer to a buffer and be reassigned to
  swap buffers. 
- A `do var` keeps its value from one run to the next.
- Many GPU-only builtins are not supportable on the CPU at orchestration time. 
  e.g. texture sampling, atomics, barriers, derivatives, `arrayLength`, 
  and `@builtin` inputs are not supported in do code.
- Do code arithmetic can be specified for more consistency than on GPUs.
- Division by zero is an error.

Not supported yet:

- Vertex shaders 
- Change to `@draw` syntax to call fragment and vertex shaders
- Return values from do blocks.
- Calling const shader functions from do blocks. 
- Operators on whole vectors or matrices. Operators work on scalars.
- Indexing with `[i]`, on buffers or on constructed arrays.
  Reading or writing buffer contents from do code is planned but not supported yet.
- Assigning struct fields and vector components.
- Bit-manipulation and packing builtins.
- createTexture()
- Public API to run do blocks from JavaScript

With [module parameters](https://github.com/webgpu-tools/wesl-spec/blob/main/proposals/222-ModuleParameters.md), do code will read parameters too. So e.g., one module parameter can set a
compute shader's `@workgroup_size` as well as the dispatch count in the do block that runs the shader.

## Examples

- [game-of-life-example](../../examples/game-of-life-example) 
- [radix-sort-do-example](../../examples/radix-sort-do-example) 
- The wgsl-play development [test page](test-page/do.html) 
