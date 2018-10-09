Extending GC Statepoints for Stackless Runtime Models
=====================================================

Introduction
------------

LLVM's Garbage Collection (GC) [Statepoints](http://llvm.org/docs/Statepoints.html) provide support for language runtime systems that use precise GC.
The primary goal of the Statepoints system is to output information alongside the generated assembly that describes where the code generator has placed live pointers in each frame of the call stack.
This information [can be used](https://github.com/kavon/llvm-statepoint-utils) by the collector of the front-end language's runtime system to identify live pointers in each stack frame during a garbage collection cycle.

The Statepoints system can go further to support runtime systems that have a "stackless" mode, where the normal call stack is not in use. A second stack, represented as a pointer value in the IR, is explicity passed to the function as an argument to be used for further function calls and to return.

Stackless models are often used to implement lightweight [green threads](https://en.wikipedia.org/wiki/Green_threads). For example, Cilk has used this type of green thread for their concurrent work-stealing runtime system [1,2].
The stackless model can also be used to efficiently implement other control-flow mechanisms derived from `call-with-current-continuation`, which is found in a number of languages like [Ruby](https://ruby-doc.org/core-2.5.1/Continuation.html) and is also [available in Boost](https://www.boost.org/doc/libs/1_68_0/libs/context/doc/html/context/cc.html) for [C++ proposal P0534R3](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0534r3.pdf).


Current Functionality
---------------------



**TODO** Example explaining how the frame **layout** and **location** are completely left up to the code generator.

The statepoint intrinsic encapsulates the main property that the GC pointer's raw value can possibly change during the call (to support moving GC), i.e., obtaining the new value from the token models this property in the IR.

The implementation of Statepoints in the code-generator lays out part of the frame for the GC pointer values so they are at known locations, and communicates that information by outputting it alongside the ASM.




Proposal
--------

We can support the stackless model by providing the ability to specify the location and partial layout of the stack frame.

**TODO: clarify the rest of this section**

The main component of this proposal are the new intrinsics
that enable the description of both the **layout** and **location** of the stack
frame to use when peforming a call, and **rules** about certian values in the
frame so that a moving garbage collector can update them.

### Function Call

Here is an example of making a function call, where we build a continuation
frame where the return address is somewhere in the middle of the frame:

```llvm

define {i64*, i64} @bar(i64*, i64) {
  ; ...
}

define void @foo(...) {

; live vals == lv0 ... lvN

; as defined by target: ptr is 8 bytes wide.
;
;                      addresses
;  low                                             high
;       -8          +8    +16    +24    .....     +72
;       | lv0 | lv1 | lv2 |  RA  |   spill area   |
;             SP
;

%buildTok = call token @llvm.frame.layout(i64* %sp,
                              i32 16,  ; RA offset
                              i32 24,  ; spill area offset
                              i32 48,  ; spill area size
                              i32 0,   ; call sp argnum
                              i32 0    ; rtrn sp argnum
                              )

; record stack frame saves
call void @llvm.frame.save(token %buildTok, i32* %lv0, i32 -8)
call void @llvm.frame.save(token %buildTok, i64* %lv1, i32 0)
call void @llvm.frame.save(token %buildTok, i64* %lv2, i32 8)
; ...
%returnTok = call token @llvm.frame.call(
                            token %buildTok,  ; frame for this call
                            {i64*, i64}(i64*, i64)* @bar, ; func
                            i64 7  ; arg(s)
                            )

%accessTok = call token @llvm.frame.accessor(token %returnTok)
%lv0_new = call i32* @llvm.frame.restore(token %accessTok, i32 -8)
; ...
%sp_new = call i64* @llvm.frame.sp(token %returnTok)
; ...
}
```

In fact, the SP is not defined to be at the top or bottom of any particular
chunk of memory, it's just a location in memory from which we define a function call frame
relative to that pointer.
The SP is always passed to the callee so that it may return back to the
caller.
To be specific, after a call, we know the following information about the
values within the frame before & after the call:

- Non-pointer values and addrspace(0) pointers are bitwise equal.
- All bits in the spill area were preserved.
- The return address's value is undefined.

Thus, addrspace(1) pointers may have been updated, i.e., by the garbage collector.

What the callee *does* with the SP is not defined, we only require
that the callee returns a pointer

Notes:
  - All offsets are relative to the location pointed to by SP.
  - We don't specify a "size" for the frame because
    that would only be needed if we were *moving* SP, but we are not.
    Once SP is captured, it does not move.
  - Any values *not* explicitly saved in the frame that are needed after
    the call will be automatically stored in the spill area. This applies to
    both explicit LLVM IR values and any values generated during codegen.
  - If the spill area is insufficient, an error is thrown.


### Function Return

At the moment, the cleanest way to handle it seems to be attaching
a partial-layout to the stack pointer.
One benefit of this approach is that it can generalize in the future to
returning values in the stack frame.

Of course, it is up to the front-end to ensure that all callers & callees
have matching layout specifications for the specific stack pointer.

```llvm
define {i64*, i64} @bar(i64*, i64) {
  ; ...
  %layoutTok = call token @llvm.frame.layout(i64* %sp, ; stack pointer
                               i32 16    ; return address offset
                               )
  %retTok = call {i64*, i64} @llvm.frame.return.TYPE(token %layoutTok, i64 12)
  ret {i64*, i64} %retTok
}
```

Note how the callee is also returning the stack pointer to its caller,
which was wrapped up in the layout token.

Prototype
---------

**TODO** link to a diff on github and explain the current state of the cpscall patch, and how this proposal is a more general version of that patch.


Related Ideas
-------------

#### Coroutines

The work on [coroutines](https://llvm.org/docs/Coroutines.html) in LLVM may appear to be similiar to this proposal, since they both deal with the concept of suspending and resuming a function, but LLVM's coroutines are different in a number of important ways.

The [coroutine lowering passes](https://llvm.org/docs/Coroutines.html#coroutine-transformation) involve splitting IR functions apart at each suspension point and inserting a return of the frame laid out by the pass.
This may work just fine for implementing coroutines, but not for general function
calls in a garbage-collected runtime system. **TODO** Problems include:

1. Lack of control over layout, and probably rules that enable moving GC.
2. Incompatible with general function calls, only ugly tricks that involve
a trampoline.
3. Inhibits optimization of the LLVM IR if we just generate pre-split code!


#### Calling Conventions

Calling conventions in LLVM can only specify the layout of function arguments
on the internal stack.
A sopisticated lowering of a calling convention can be used to implement many
parts of this proposal (not all), however, it would need to be done specifically
for one LLVM front-end.

Notes
-------

This `README` represents a plain-text summary of the full paper found under
the `paper` directory.


References
----------

1. Blumofe, Robert D., et al. "Cilk: An efficient multithreaded runtime system." Journal of Parallel and Distributed Computing 37.1 (1996): 55-69.
2. Frigo, Matteo, Charles E. Leiserson, and Keith H. Randall. "The implementation of the Cilk-5 multithreaded language." ACM Sigplan Notices 33.5 (1998): 212-223.
