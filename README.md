
Front-end Controlled GC Statepoints
======

Introduction
------

LLVM's Garbage Collection (GC) [Statepoints](http://llvm.org/docs/Statepoints.html) provide support for language runtime systems that use precise GC.
The primary function of the Statepoints system is to output information alongside
the generated assembly that describes where LLVM has placed live pointers in
each stack frame it has laid out.
This information [can be used](https://github.com/kavon/llvm-statepoint-utils) by the collector of the front-end language's
runtime system to identify live pointers in each stack frame during a
garbage collection cycle.

With some upgrades, the Statepoints system can go further to support many language runtime systems that have advanced features:

1. Some operate in a "stackless" mode, where the internal call stack is not in use, and instead a second stack that is passed to the function as an argument is used for calls. This technique is often used to implement lightweight [green threads](https://en.wikipedia.org/wiki/Green_threads).

2. Some runtime systems have special layouts for their call stacks, for example, to support custom exception-handling mechanisms or a lazy evaluation strategy.


Proposal
------

Here is an example of building a frame where the return address is somewhere
in the middle of the frame:

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
array, it's just a location in memory from which we define a function call frame.
The SP is always passed to the callee so that it may return back to the
caller.
Every callee G must also include as a return value a pointer to a frame
that is equivalent to the one the caller F had originally passed to G.
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
