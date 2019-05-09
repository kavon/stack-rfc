Extending LLVM for Stackless Runtime Models
=====================================================

Introduction
------------

LLVM be extended to support runtime systems that offer a
"stackless" mode, where the normal call stack is not in use.
A second stack, allocated by the language's runtime system, is
explicitly passed to a function as an argument to be used to allocate
frames capturing the continuation of further function calls and to return to
the caller.
Stackless execution models are often used to implement efficient
[green threads](https://en.wikipedia.org/wiki/Green_threads) and stackful coroutines.

A language front-end that utilizes this compilation model faces a challenge in using
LLVM due to LLVM IR's lack of expressiveness in terms of
calling conventions with respect to call stacks.
This proposal aims to remedy this situation without radical changes to LLVM IR.

**TODO: conclude with a better summary of what this proposal is about, etc.**



Proposal
--------

To support functions operating in a "stackless" dicipline,
there are two main requirements.

#### Stack Location

We need the ability to specify the
*location* of the stack frame to be used by the stackless function.
We cannot use the existing stack that
LLVM implicitly allocates and manages in the standard register for that platform,
since in a stackless model that stack should not be used.
Instead, we have a second "stack"
(which is typically heap allocated) that must be used for the function's
frame instead.

While a custom prologue/epilogue can be used to implement different strategies
for allocating stack space (e.g., segmented stacks), the register used to hold
the stack pointer appears to be hardwired for each target achitecture in LLVM,
and thus not customizable for each calling convention on that architecture.
Thus, I'm proposing changes to calling convention specification such that

**EITHER**

1. an "alternate" stack register can be defined, and if present, used for that
function. the existing primary stack is assumed to still exist with standard
properties (i.e., alignment, growth direction), we just don't allocate a frame there.

2. the primary stack's register is customizable. if a stack using the old register
   will still be hanging around, that register should be marked as reserved / unused.

Goal is to have calls from an alt-function to a standard function
work seamlessly. Question is, how do we
pass our alternate stack to the callee for, e.g., gc purposes?
In addition, how does an alt-function return to a standard function?
How does a standard function call an alt-function.
Perhaps we need to have a `@llvm.stackpointer()` intrinsic to reify the value for the sole purpose of passing it in an argument register to the callee?

We might need multiple calling conventions on functions to make the transitions
to and from a primary/secondary stack operation work.
The primary reason for this is that a calling convention in LLVM defines both its
call and return convention.

```
case (caller function, call instruction, callee function) of

(X, Y, Z) if X = Y = Z
                  => easy case.

(std_cc, alt_enter_cc, alt_enter_cc) =>
               all callers of the callee pass an alt stack in the first
               argument register of this convention.
               This argument register uses the same register as the alt-stack
               in alt_cc.
               the caller's continuation is saved in the primary
               stack.
               Thus, returns from this function are always performed on std-stack.

(alt_enter_cc, alt_cc, alt_cc) =>
               the caller's continuation is saved on the _alternate_
               stack and passed to the callee.


(alt_cc, alt_leave_cc, alt_leave_cc) =>
              the caller saves its continuation to the alt stack.
              the callee returns via the alt stack.
              otherwise, the callee allocates its continuation(s) on the
              primary stack.

(alt_cc, std_cc, std_c) =>
              the caller's continuation is saved on the alt stack,
              and the primary stack is prepared to resume control
              from the callee.
              The callee returns using the primary stack.

```


<!--
##### Challenges

The primary challenge is that we need an understanding of how
two functions with different calling conventions with respect to stack-passing
can interoperate without a separate assembly-code shim.

TODO: can this even work without an assembly shim function
that unifies the conventions, whether auto-generated or not?

If the caller function F's convention has a different stack convention from
a call to the callee G, then:

  1. The stack register of F must be a callee-saved register in G.
  2. The stack register of G must be an argument register in F.

During codegen, we lower the call such that the caller initalizes ???

This way, a C-convention function can allocate a fresh stack and pass it to
a stackless function so that it can execute on it, and

We can either leave this undefined, or check for this and reject
such programs. We can leave it up to the implementor of the calling
convention to identify which conventions are compatible, instead of
an automated test / check.
-->



#### Stack Layout

For garbage-collected runtimes, we need the ability to partially
control the *layout* of objects within the frame so that this second
stack can be parsed and used by the runtime system and code generated by the
frontend (i.e., a custom exception-handling mechanism).
Specifying where GC pointers should be placed is
the primary requirement.

The GC Statepoints infrastructure
[already has undocumented support for custom layouts](https://github.com/llvm/llvm-project/commit/f8f0933b488bcbd1cc4b69a93bb18f8c33ce6847)
that are specified via allocas, similar to the less-maintained GC Root infrastructure.
In fact, this feature was originally intended as the start of reimplementing GC Root
in terms of GC Statepoints.

The other values that may live on the stack, like the call's return
address and any register spills generated, are not real values in LLVM IR.
Thus, it is up to the code generator to determine where these values
will be placed.
For control over these values, we can add custom lowering that can be performed
per GC Strategy or calling convention for GC Statepoints.

The benefit of offering custom lowering for Statepoints is that it would finally
allow us to unify GC Root and GC Statepoint to offer one comprehensive and robust
system for GC.


Examples
--------

**TODO: rewrite this in terms of alloca-based layout, i.e., alloca order
in the function and alignment define the layout of part of the stack frame,
and a store to an alloca followed by passing a pointer to that alloca
to the statepoint intrinsic means that's where you want the value
to live on the stack. See f8f0933b488bcbd1cc4b69a93bb18f8c33ce6847**

<!--

#### Function Calls

Let's consider our first example, where `@foo` calls `@bar`, written for a
"stackless" model. Both `@foo` and `@bar` now accept and return a secondary
stack, `%sp`, as their first argument when called, and the first structure
member on return.

In addition, we need to specify a layout for the frame to be compatible with our
runtime system: from low to high addresses we have GC pointers, the return
address, non-GC values, and then any register spills. This can be achieved using
the proposed `gc.frame.*` intrinsics below:

```llvm
%stack_ty = type i64*
declare %stack_ty @AllocHeap(i32)
declare {%stack_ty, i32} @bar(%stack_ty, i32)

define {%stack_ty, i32} @foo(%stack_ty %sp_arg, i32 %arg) {
  ;;; ...
  %ptr = ; an i64*, tracked by the GC
  %raw = ; an i32

  ;;; NEW: allocate space for the frame
  %spill_sz = call i32 @llvm.gc.frame.spillsz()
  %frame_sz = add i32 %spill_sz, 32
  %sp = call %stack_ty @AllocHeap(i32 %frame_sz)

  ;;; NEW: specify a layout for the frame
  %frameTok = call token @llvm.gc.frame.layout(
        %stack_ty %sp,  ; where to allocate the stack frame
        i32 16,         ; byte offset for return address
        i32 32          ; byte offset for start of spill area
    )

  ;;; NEW: record frame writes at specified byte offsets
  call void @llvm.gc.frame.save(token %frameTok, %stack_ty %sp_arg, i32 0)
  call void @llvm.gc.frame.save(token %frameTok, i64* %ptr, i32 8)
  call void @llvm.gc.frame.save(token %frameTok, i64* %raw, i32 24)

  ;;; perform the call @bar(42)
  %tok = call token @llvm.gc.frame.statepoint(
             i32 (i32) @bar,   ; callee
             i32 42,           ; arg(s)
             token %frameTok   ; frame layout
             )

  ;;; NEW: %sp and %frameTok are now invalid since the GC might have
  ;;; relocated the frame. we obtain the new version.
  %relo_frameTok = call token @llvm.gc.frame.accessor(token %tok)

  ;;; %ptr is now invalid. must use relocated version. NEW: now a byte offset
  %relo_ptr = call i64* @llvm.gc.frame.load(token %relo_frameTok, i32 8)

  ;;; obtain the value(s) returned by @bar
  %retVal = call i32 @llvm.gc.frame.result(token %tok)

  ;;; ... uses of %raw, %relo_ptr, etc ...
}
```

Note that the primary changes here are an expansion of certain aspects that were
previously left undefined in `gc.statepoint` to allow more control by front
-ends:

(1) `gc.frame.layout` wraps the pointer to the frame (i.e., the location) along
with some layout requirements for the non-representable values (the return
address and spill area).

(2) The use of `gc.frame.save` corresponds to listing the live GC pointers in
`gc.statepoint` calls so they are saved to the frame, however, now we can
control the layout within the frame. Similarly, `gc.frame.load` corresponds to
Statepoint's `gc.relocate` to freshly load the values that were saved to the
frame.

Just as in `gc.statepoint`, any values that are used after the call to `@bar`
that are *not* explicitly stored and reloaded from the frame will be
automatically stored and reloaded from the frame's spill area. This applies to
both explicit LLVM IR values and any values generated during code generation.
The size of this spill area will be filled in by the code generator via the
`gc.frame.spillsz` intrinsic. As usual, it is assumed that the garbage collector
will not modify the spill area or the return address.


#### Function Return

Let's now consider how `@bar` returns back to `@foo`. We again use the
`gc.frame.layout` intrinsic to describe where the return address is located
within the the frame `@bar` was given. It is up to the front-end to ensure that
all frame layouts for callers & callees have matching layout specifications.

```llvm
define {%stack_ty, i32} @bar(%stack_ty %sp_arg, i32) {
  ;;; ...
  %layoutTok = call token @llvm.gc.frame.layout(
                               %stack_ty %sp_arg, ; stack pointer
                               i32 16             ; return address offset
                               )
  %retTok = call {%stack_ty, i64} @llvm.gc.frame.return(token %layoutTok, i32 12)
  ret {%stack_ty, i32} %retTok
}
```

Note that only a partial layout is required for returning, as the rest of the
frame is not needed by `@bar`. The vararg `gc.frame.return` intrinsic to bundles
up the frame layout along with the returned values. In this example, 12 will be
returned as the 2nd struct member, as the 1st member is always the stack
pointer. It is also possible to have a value that is returned via the stack,
though we are not pursuing this yet.

#### Green Threads

**TODO: example IR for this**

#### Coroutines

**TODO: example IR for this**

-->

Patch-set Plan
--------------

The changes proposed here can be organized into 3 easy-to-review patches:

- A patch that adds the ability to control which register is used for
  the stack / frame pointer in a given calling convention.

- A patch that improves and documents the alloca-based stack frame layout
  for roots in GC Statepoint.

- A patch that offers custom lowering of GC Statepoint pseudo-instructions
  in Machine IR.
