Operating on a Secondary Stack
==============================

Major implementations of Haskell (GHC) and Standard ML (SML/NJ) utilize a
"stackless" runtime model, where the primary call stack is not in use. A second
stack, often allocated in the heap, is passed to a function that uses it to
allocate its frame for further function calls, etc. The primary stack is also
passed to a stackless function, but it is only used for efficient foreign
function calls since those must comply with the system ABI instead of the
language's ABI.

Stackless execution models can also be used to implement efficient green threads
[1] and stackful coroutines [2] (which are not supported by LLVM's coroutines),
because usage of the secondary stack is extremely efficient. A coroutine or
thread in a stackless model yields control by calling a function that operates
on the primary stack, passing to it the secondary stack as an argument.


Problem
-------

The major difficulty in supporting a "stackless" runtime model in LLVM is that
calling conventions have no control over which register holds the stack pointer.
Currently, (at least for x86) its up to each target to define the stack pointer
register and its conventions (e.g., growth direction, alignment).

Much of the allocation & alignment of the stack pointer can be controlled by
defining new prologue-epliogue insertion implementations as other languages have
done in X86FrameLowering. Further adjustments for specific operating system
or language ABIs are also done here and in other places.

It seems that the purpose of X86RegisterInfo::getStackRegister() is
to provide an abstraction for the physical stack pointer register, but in many
parts of the x86 code generator, the standard stack pointer register
(RSP) is hardcoded into the implementation.


Question
--------

My primary question is: what is the best way to allow for an additional stack
pointer register that is used for frame allocation?

My initial thought was to define a new subtarget and override relevant parts of
the code generator so a different register is used to allocate frames.
If a subtarget is the right approach, is there a way to implement a
subtarget as a plugin or should it be part of the main branch?


Links
-----

[1] https://en.wikipedia.org/wiki/Green_threads
[2] https://www.boost.org/doc/libs/1_57_0/libs/coroutine/doc/html/coroutine/intro.html#coroutine.intro.stackfulness
