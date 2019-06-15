# Current Status

In the original implementation, we make the assumption that LLVM's code generator will not need to generate stack spills since nothing is live across the call by construction. In fact, the code generator can and will generate (sometimes useless) spills that must be stored in the heap-allocated stack frame [see BUGS].

Thus, since the existing design of the CPS call instrinsic in LLVM IR is spiritually the same as LLVM's GC Statepoints, I've been investigating ways to extend Statepoints for our purposes, and I think I've found a clean way forward thanks to two factors: (1) the introduction of "callbr" in LLVM, and (2) a discovery I made last month about Statepoints.

## Factors

(1)  CallBr is a new terminator for basic blocks in LLVM IR that performs a function call and specifies one or more successor blocks upon returning, just as in Cmm. This new instruction allows the block address of the successor to be reified as a value and passed to the callee, such as for the purposes of implementing setjmp in C. 

The great part of CallBr is that it solves one of the annoyances in the the CPS call design, namely, the need for a pre-ISel "prep" pass to ensure the CPS calls were at the end of the blocks [see Reference 1]. In addition, it should allow us to directly save the block address that the call will return to right into memory! This was bundled up with the CPS call instrinsic, which required the offset to write the return address into when making the call.

What's missing however is a corresponding "retbr", which takes a code address and performs a jump to return to one of the successors of the caller's "callbr". This is something that we'll need to propose as a means to implement longjmp.

Further information about callbr:

Appearance in documentation as of now:
http://llvm.org/docs/LangRef.html#callbr-instruction

Introduction to CallBR:
https://groups.google.com/d/msg/llvm-dev/tq1KpUsOsBU/4id2FfMTBgAJ


(2)  With Callbr in hand, we primarily need a way to describe the layout of heap-allocated stack frames.
The GC Statepoints infrastructure already has undocumented support for custom layouts
that are specified via allocas, similar to the less-maintained GC Root infrastructure.
Essentially, a pointer to an alloca is treated as a stack frame slot that contains a live pointer by Statepoints.
Since allocas are allocated on the stack in-order and respect alignments specified by the front-end.

Pros of using Statepoints? (a) They're well-understood by LLVM's optimizer 
(b) have other dedicated users (namely, the Falcon system for Java by Azul Systems) that work on the feature occasionally
(c) They're already integrated with LLVM's frame allocation system.

Any spills that happen to pop-up during code generation will end up being placed in the frame. 
The extensions we need to make in the lowering of stack frames in LLVM is that the stack-pointer register is not
the system-default, since that contains the foreign-function stack pointer.

Next, we need to design a new prologue-epilogue insertion specific to GHC that bump-allocates a new stack frame on entry
in the heap.
This prologue should include a heap-limit test.  
I've done this for Manticore for heap-allocated stack frames (this is as-of-yet unpublished work, but it works just fine for us).

As far as I can tell, it's not possible to write a dynamic LLVM plugin for the prologue-epilogue insertion pass, but this should
not be hard to add.

Cons? Statepoints are currently only implemented for x86-64. We will want to implement it for ARM too.


### Writing

Question I've been meaning to ask about the stack pointer register: https://github.com/kavon/stack-rfc/blob/master/QUESTION.md
The reason for this question is that I think we might want to define a "GHC" vendor subtarget plug-in to override the lowering of Statepoints so they are compatible with our runtime system.

A design document that has not yet been updated with the new plans: https://github.com/kavon/stack-rfc/blob/master/README.broad.md


### Bugs

These were found in the initial implementation. The workarounds are essentially disabling certain optimizations, which is not sustainable:

Bug 1. https://github.com/kavon/ghc-llvm/issues/11
Bug 2. https://github.com/kavon/ghc-llvm/issues/12
Bug 3. https://github.com/kavon/ghc-llvm/issues/20

### References

1. https://github.com/kavon/ghc-llvm/issues/11#issuecomment-308133107
