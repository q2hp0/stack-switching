
> [!WARNING]
> **Important:** Everything in this project is mainly for research, experimentation, and learning. I built it to understand how stack switching, execution contexts, coroutines, and low-level CPU/ABI behavior actually work underneath the abstractions.
>
> Don't take the implementation as a production-ready runtime or as a recommendation for how these systems should be built in real-world software. Some parts are intentionally simplified, experimental, or unsafe because the main goal here is learning by actually implementing and testing the concepts myself.
>
> Basically, this project exists because I wanted to understand **what is actually happening under the hood**, not because I was trying to create a production coroutine runtime.
>



# Cooperative Stack Switching on x86-64

A small low-level learning project about **stack switching, execution contexts, and cooperative coroutines on x86-64**.

The main idea is simple:

https://q2hp0.github.io/stack-switching/

> What actually happens when you stop executing on one stack and continue on another one?

Instead of starting with a coroutine library or a high-level abstraction, this project looks at the lower level:

- CPU registers
- `rsp`
- stack frames
- `call` / `ret`
- saved execution state
- callee-saved registers
- manually created stacks
- forged return addresses
- context switching
- cooperative scheduling
- stack alignment
- guard pages
- signals
- CET
- unwinding
- generators
- stackful vs stackless execution

This is mainly an **experimental / educational environment**.

It is not intended to be a production coroutine runtime, threading library, or replacement for mature runtimes.

---

## The basic idea

A running execution context can be thought of roughly as:

```text
context = CPU state + reachable stack state
```

For cooperative stack switching, the important part is usually the stack pointer:

```text
rsp
```

A very simplified context switch looks like:

```asm
mov [old_context], rsp
mov rsp, [new_context]
ret
```

Real implementations also need to preserve the registers required by the calling convention.

The interesting part is that `ret` does not know anything about coroutines.

It simply reads an address from the new stack and continues execution there.

That means a new execution context can be created by preparing a stack that looks like a valid suspended call frame.

---

## What this project is trying to show

The project focuses on the mechanics rather than hiding them behind an API.

You can see how:

```text
normal function call
        |
        v
     stack frame
        |
        v
   saved execution state
        |
        v
    switch stack
        |
        v
       ret
        |
        v
different execution path
```

This is basically the foundation behind many forms of cooperative execution.

---

## Stackful execution

A stackful coroutine has its own execution stack.

That means it can suspend from relatively deep inside a normal call chain.

For example:

```text
coroutine
  |
  +-- function_a()
        |
        +-- function_b()
              |
              +-- function_c()
                    |
                    +-- yield()
```

The coroutine does not necessarily have to return manually through every function.

The stack itself contains the suspended call chain.

After switching back later, execution can continue from that suspended state.

---

## Stackless execution

Stackless coroutines use a different model.

Instead of keeping a complete independent call stack, the compiler or runtime usually creates a state object containing the information needed to continue.

Conceptually:

```text
coroutine frame
    |
    +-- state
    +-- locals
    +-- instruction/state position
    +-- suspended values
```

This can be cheaper in some situations, but suspension generally happens at known points.

---

## Different stack models

| Model | Where state lives | Suspension |
|---|---|---|
| Stackful | Separate execution stack | Can happen deep inside normal calls |
| Stackless | Compiler-generated coroutine frame | Usually explicit suspension points |
| Segmented stack | Multiple stack segments | Can grow dynamically |
| Copying stack | Stack contents moved between locations | Requires careful pointer handling |
| Shared/copying stack | One stack copied on switch | Cost depends on live stack depth |

Another useful way to look at them:

| Model | Main idea | Main trade-off |
|---|---|---|
| Stackful | Each execution context owns a stack | More memory |
| Stackless | State is stored in a frame | More explicit state handling |
| Segmented | Stack grows through segments | More runtime complexity |
| Copying | Save/restore stack contents | Copying cost |
| Shared/copying | Reuse stack storage | Switch cost depends on live data |

---

# x86-64 registers

On System V AMD64, some registers are preserved by the called function and others are not.

The important callee-saved registers are:

```text
rbx
rbp
r12
r13
r14
r15
```

The stack pointer is:

```text
rsp
```

Other commonly used caller-saved registers include:

```text
rax
rcx
rdx
rsi
rdi
r8
r9
r10
r11
```

A minimal cooperative context switch therefore normally needs to preserve at least the state required by the ABI.

---

# A minimal context switch

A simplified NASM-style implementation can look like this:

```asm
ctx_switch:
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15

    mov [rdi], rsp
    mov rsp, rsi

    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp

    ret
```

The important operations are:

```asm
mov [rdi], rsp
mov rsp, rsi
ret
```

Everything around them exists to make the transition compatible with the ABI and preserve the expected registers.

---

# Why `ret` is enough

`ret` roughly does:

```text
address = [rsp]
rsp += 8
jump address
```

So if the new stack contains:

```text
+-------------------+
| return address    | <-- rsp
+-------------------+
| stack data        |
| stack data        |
| stack data        |
+-------------------+
```

then:

```asm
ret
```

can transfer execution to that address.

This makes manually constructed coroutine contexts possible.

You do not need a special CPU instruction called:

```text
switch_to_coroutine
```

The CPU already provides the basic primitives.

---

# Forging a context

A new stack can be prepared before the coroutine starts.

Conceptually:

```text
high address
+----------------------+
|                      |
|      stack space     |
|                      |
+----------------------+
| saved registers      |
+----------------------+
| initial return addr  |
+----------------------+ <-- rsp
low address
```

When the context switch loads this `rsp` and executes:

```asm
ret
```

the CPU starts executing at the prepared return address.

This is one of the most interesting parts of the project.

The coroutine is not magically created.

Its initial execution state is manually constructed.

---

# The stack is part of the state

For a stackful coroutine, the stack contains things such as:

```text
return addresses
local variables
saved registers
temporary values
function frames
compiler-generated data
```

So switching the stack pointer can effectively switch the execution history that is reachable from that stack.

A simplified model is:

```text
C = (sp, S)
```

where:

```text
sp = current stack pointer
S  = stack storage
```

The exact machine state is more complicated, but this model is useful for understanding the idea.

---

# Cooperative scheduling

Once multiple contexts exist, a scheduler can choose between them.

For example:

```text
context A
context B
context C
context D
```

A simple scheduler could do:

```text
A
 |
yield
 |
B
 |
yield
 |
C
 |
yield
 |
D
 |
yield
 |
A
```

The scheduler does not need to understand the internal call chain of each coroutine.

It mainly needs to know:

```text
which context is runnable
where its stack is
where its saved state is
```

---

# A tiny scheduler model

A conceptual scheduler might look like:

```c
while (running) {
    context_t *next = scheduler_next();

    switch_context(
        &current->stack_pointer,
        next->stack_pointer
    );

    current = next;
}
```

Real implementations need more bookkeeping.

The important idea is that the scheduler changes the current execution context instead of creating an operating-system thread for every task.

---

# Cooperative vs preemptive

This project focuses primarily on cooperative switching.

Cooperative:

```text
task
 |
 | decides to yield
 v
scheduler
 |
 v
another task
```

Preemptive:

```text
task
 |
 | CPU keeps executing
 |
interrupt/signal
 |
 v
scheduler
```

Cooperative switching is much easier to reason about because the running code chooses when the transition happens.

---

# Why cooperative switching is interesting

A normal OS thread has a relatively expensive abstraction around it:

```text
thread
  |
  +-- kernel scheduling
  +-- kernel stack
  +-- thread state
  +-- synchronization
  +-- scheduler interaction
```

A user-space coroutine can be much smaller:

```text
coroutine
  |
  +-- stack
  +-- saved registers
  +-- scheduler state
```

This does not make coroutines automatically better.

It just means the execution model is different.

---

# Stack allocation

A coroutine needs somewhere to run.

A simple implementation can allocate memory for a stack using `mmap`.

Conceptually:

```c
void *stack = mmap(
    NULL,
    stack_size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1,
    0
);
```

The stack grows toward lower addresses on normal x86-64 systems.

So the initial stack pointer is normally near the high end of the allocation.

For example:

```text
high address
+----------------------+
| initial rsp          |
|                      |
|                      |
| usable stack         |
|                      |
+----------------------+
low address
```

---

# Guard pages

A guard page can be placed next to the usable stack.

Conceptually:

```text
+----------------------+
| usable stack         |
|                      |
|                      |
+----------------------+
| PROT_NONE guard page |
+----------------------+
```

If the stack grows into the guard page, the process gets a memory fault instead of silently corrupting unrelated memory.

This is especially useful for experiments with manually allocated stacks.

---

# Stack alignment

The System V AMD64 ABI has stack alignment requirements.

This matters.

A context switch that leaves:

```text
rsp
```

at the wrong alignment can cause problems later, especially when normal compiled functions expect ABI-compliant alignment.

The switch itself may appear to work:

```text
switch()
ret
```

but some unrelated function can later fail because the stack is malformed.

So stack construction has to account for alignment.

---

# The red zone

System V AMD64 also defines a 128-byte red zone below `rsp`.

Leaf functions may use this area without adjusting the stack pointer.

That matters when asynchronous events such as signals are involved.

A context-switching runtime that does not account for the red zone can corrupt data that normal compiled code assumes is temporarily safe.

This is one reason low-level stack manipulation gets complicated quickly.

---

# Signals

Signals introduce another layer.

A signal can interrupt execution asynchronously.

That means the program might be interrupted while:

```text
inside function_a()
    |
    +-- function_b()
          |
          +-- function_c()
```

and the stack may contain temporary state that was never intended to be manually copied or moved.

Signal handlers can also use an alternate signal stack.

That makes signal-based execution switching possible, but it introduces a lot more rules.

---

# CET and shadow stacks

Modern x86-64 systems may have Control-flow Enforcement Technology (CET).

One part of CET is the shadow stack.

Normally:

```text
normal stack
    |
    +-- return address
```

With shadow stacks, return addresses are also protected separately.

Conceptually:

```text
normal stack
    |
    +-- return address

shadow stack
    |
    +-- protected return address
```

A context switch that simply changes:

```asm
rsp
```

may not be enough when shadow-stack protection is active.

That is one of the reasons old stack-switching tricks cannot always be assumed to work unchanged on modern systems.

---

# Unwinding

Debuggers and exception/unwind mechanisms expect stack frames to make sense.

A manually switched stack can look strange to:

```text
gdb
debuggers
profilers
unwinders
sanitizers
crash handlers
```

Without correct unwind metadata, the runtime may work while debugging information becomes unreliable.

For production-quality runtimes, stack switching and unwinding need to be designed together.

---

# Context invariants

A valid context should satisfy a few basic assumptions.

The stack should contain the expected frame shape.

The saved registers should be restored in the same order they were saved.

The stack pointer should point to the expected location.

The return address should be valid.

The stack should belong to the context being restored.

Conceptually:

```text
save order
    =
restore order
```

and:

```text
saved rsp
    |
    v
expected stack layout
    |
    v
valid return address
```

Breaking any of these assumptions can produce very strange crashes.

---

# Multiple contexts

With multiple contexts:

```text
Context A
    |
    +-- stack A
    +-- saved rsp A

Context B
    |
    +-- stack B
    +-- saved rsp B

Context C
    |
    +-- stack C
    +-- saved rsp C
```

Switching from A to B means:

```text
save rsp A
    |
    v
load rsp B
    |
    v
restore B state
    |
    v
ret
```

Later B can switch back to A:

```text
save rsp B
    |
    v
load rsp A
    |
    v
restore A state
    |
    v
ret
```

---

# Yield

A simple coroutine API could conceptually expose:

```c
yield();
```

Internally it could perform something like:

```text
current context
       |
       v
save current rsp
       |
       v
scheduler chooses next
       |
       v
load next rsp
       |
       v
restore registers
       |
       v
ret
```

The API is simple.

The implementation underneath is not.

---

# Generators

The same mechanism can be used to build generators.

For example:

```text
generator
    |
    +-- produce value
    |
    +-- yield
    |
    +-- resume
    |
    +-- produce value
    |
    +-- yield
```

The generator keeps its execution stack between calls.

This means local variables and nested function calls can remain alive across a yield.

---

# Stackful vs stackless generators

A stackful generator can conceptually suspend here:

```text
generate()
    |
    +-- parse()
          |
          +-- decode()
                |
                +-- yield()
```

A stackless implementation normally needs the compiler/runtime to represent the suspended state explicitly.

Something like:

```text
state = DECODE
value = ...
```

Then resume continues based on that state.

Both approaches are useful.

They simply represent suspended execution differently.

---

# Segmented stacks

A segmented stack does not necessarily reserve one huge contiguous region.

Instead:

```text
segment A
    |
    v
segment B
    |
    v
segment C
```

When more stack space is needed, another segment can be added.

This can reduce unused reserved memory, but crossing segment boundaries adds complexity.

---

# Copying stacks

Another approach is to use a stack area and copy its live contents when switching.

Conceptually:

```text
running stack
+----------------+
| live frames    |
| live data      |
+----------------+
        |
        | copy
        v
saved stack
+----------------+
| live frames    |
| live data      |
+----------------+
```

When the context resumes, the stack can be copied back.

The obvious cost is proportional to the amount of live stack data.

---

# Shared / copying stacks

Instead of giving every coroutine a permanent large stack:

```text
Coroutine A -> stack
Coroutine B -> stack
Coroutine C -> stack
Coroutine D -> stack
```

a runtime can reuse stack storage.

For example:

```text
shared stack
     |
     +--> A
     |
     +--> B
     |
     +--> C
```

When switching, the current stack contents may need to be preserved.

This saves memory but introduces copying and pointer-management problems.

---

# Pointer problems

Moving a stack is not as simple as:

```c
memcpy(destination, source, size);
```

A stack may contain pointers to objects located inside that stack.

For example:

```text
stack
+----------------------+
| local object         |
| pointer ------------+------+
+----------------------+
```

After moving the stack:

```text
old address
     ^
     |
pointer
```

the pointer may still reference the old location.

That is one of the major complications of copying-stack designs.

---

# Intrusive scheduling

A scheduler can keep runnable contexts in a queue.

For example:

```text
+-----+    +-----+    +-----+    +-----+
| A   | -> | B   | -> | C   | -> | D   |
+-----+    +-----+    +-----+    +-----+
   ^                                      |
   +--------------------------------------+
```

A circular run queue makes simple round-robin scheduling possible.

Conceptually:

```text
A -> B -> C -> D -> A
```

A yield can move the current context to the back of the queue.

---

# Round-robin

A very small scheduler can use:

```text
A
B
C
D
A
B
C
D
```

No priorities.

No work stealing.

No complex scheduling policy.

Just:

```text
next runnable context
```

This is enough to demonstrate the mechanics.

---

# Blocking

Cooperative runtimes also need to think about blocking.

If one coroutine performs a blocking operation:

```text
coroutine A
    |
    +-- blocking syscall
```

the whole underlying thread may stop making progress.

A coroutine scheduler therefore often needs non-blocking I/O or a mechanism that prevents one coroutine from blocking the entire worker thread.

---

# One thread, many contexts

The model can be:

```text
one OS thread
       |
       +-- coroutine A
       +-- coroutine B
       +-- coroutine C
       +-- coroutine D
```

Only one context is executing at a time.

The scheduler controls which one owns the CPU.

This is different from:

```text
four OS threads
```

where the kernel schedules each thread independently.

---

# Context switching is not magic

At the lowest level, a context switch is mostly about state.

Something like:

```text
old context
    |
    +-- save registers
    +-- save rsp
    |
    v

new context
    |
    +-- load rsp
    +-- restore registers
    +-- continue execution
```

The difficult part is not the number of assembly instructions.

The difficult part is making every surrounding assumption correct.

---

# Things that can break

A low-level stack switch can fail because of:

```text
wrong rsp
wrong stack alignment
wrong return address
missing register
bad stack lifetime
stack overflow
guard-page mistakes
signal interaction
red-zone corruption
unwinding problems
CET shadow stacks
ABI violations
invalid pointer relocation
```

A crash does not necessarily happen inside the context-switch function.

It can happen much later.

---

# ABI matters

The compiler assumes the ABI is respected.

If the assembly violates those assumptions, normal C code can behave incorrectly.

For example:

```text
assembly
   |
   +-- switch context
   |
   v
C function
   |
   +-- uses ABI assumptions
   |
   v
unexpected crash
```

This is why context-switching assembly should be kept small and predictable.

---

# Why assembly is used

A normal C function cannot portably say:

```c
rsp = another_stack;
```

The compiler controls the machine-level representation.

Assembly lets us explicitly manipulate:

```text
rsp
rbp
rbx
r12
r13
r14
r15
```

and perform the final:

```asm
ret
```

instruction.

This project uses assembly specifically because that is where the interesting part actually happens.

---

# Architecture

The project is primarily focused on:

```text
Linux
x86-64
System V AMD64 ABI
NASM
C
```

The same general idea exists on other architectures, but the actual registers and calling conventions change.

---

# AArch64

On AArch64, the mechanism is different.

There is no direct equivalent of simply relying on the x86-64 `ret` pattern in exactly the same way.

A context implementation needs to deal with:

```text
sp
x19-x28
x29
x30
```

where:

```text
x30 = link register
```

The return mechanism is different from x86-64.

The general idea is still the same:

```text
save execution state
        |
        v
save stack pointer
        |
        v
load another context
        |
        v
restore state
        |
        v
continue execution
```

---

# What is intentionally simplified

This project does not try to implement every feature required by a serious production runtime.

Some parts are intentionally simplified:

```text
simple scheduler
simple stack allocator
minimal context structure
limited error handling
minimal ABI assumptions
cooperative switching
experimental assembly
```

The goal is to make the mechanism understandable.

Not to build another Go runtime.

---

# What this is NOT

This is not:

```text
a production coroutine library
a threading library
a replacement for pthreads
a replacement for Go's runtime
a replacement for async runtimes
a hardened stack-switching runtime
a portable coroutine implementation
```

It is a low-level experiment for understanding how these systems can work.

---

# Building

On Debian/Ubuntu-style systems:

```bash
sudo apt install build-essential nasm
```

A minimal build can look like:

```bash
nasm -f elf64 ctx.asm -o ctx.o
gcc -no-pie main.c ctx.o -o ctx
./ctx
```

The exact build commands may change depending on the files in the project.

---

# Example execution flow

A very simplified run can be thought of as:

```text
main
 |
 v
create stack
 |
 v
prepare context
 |
 v
switch to context
 |
 v
coroutine starts
 |
 v
yield
 |
 v
scheduler
 |
 v
main / another coroutine
 |
 v
resume
 |
 v
coroutine continues
```

Nothing about this requires the coroutine to be a special CPU feature.

It is mostly normal machine state arranged in a useful way.

---

# The interesting part

The most important idea in this project is probably this:

```text
A suspended computation can be represented by its saved execution state
and the stack that contains the rest of its call chain.
```

Once that idea makes sense, a lot of other concepts become easier to understand:

```text
coroutines
generators
user-space schedulers
fibers
green threads
cooperative multitasking
stackful execution
context switching
```

They are different abstractions built around related mechanisms.

---

# Stackful execution in one picture

```text
OS thread
    |
    +-------------------------------+
    |                               |
    |     scheduler                 |
    |         |                     |
    |         +----> coroutine A    |
    |         |         |           |
    |         |         +-- stack A |
    |         |                     |
    |         +----> coroutine B    |
    |         |         |           |
    |         |         +-- stack B |
    |         |                     |
    |         +----> coroutine C    |
    |                   |           |
    |                   +-- stack C |
    |                               |
    +-------------------------------+
```

Only one coroutine is executing at a time, but each one has its own suspended call chain.

---

# The smallest mental model

If all of this feels like too much, the basic model is just:

```text
1. Allocate a stack.

2. Put a valid starting address on it.

3. Save the current rsp.

4. Load the new rsp.

5. Restore the required registers.

6. ret.

7. The other execution context is now running.
```

Then later:

```text
1. Save its rsp.

2. Load the old rsp.

3. Restore the old registers.

4. ret.

5. Continue where the old context stopped.
```

That is the core idea.

Everything else exists because real software has to deal with ABI rules, memory safety, debugging, signals, modern CPU security features, and different architectures.

---

# Why this project exists

High-level coroutine APIs make this stuff look easy.

For example:

```text
spawn()
yield()
resume()
```

But underneath those APIs there still has to be some representation of:

```text
where execution stopped
what registers need to survive
what stack contains the suspended calls
where execution should continue
```

This project is mainly about removing that abstraction and looking at those pieces directly.

---

# Experimental status

This repository is intentionally an experiment.

The code may use low-level techniques that are perfectly useful for learning but are not necessarily appropriate for production software.

Do not treat the implementation as a drop-in runtime.

The interesting part is understanding why it works, what assumptions it makes, and where those assumptions stop being valid.

---

# Possible directions

There are a lot of things that can be experimented with after the basic switch works:

```text
multiple coroutines
round-robin scheduler
generators
channels
timeouts
sleep queues
I/O integration
guard pages
stack pooling
stack growth
stack copying
segmented stacks
signal-based preemption
alternate signal stacks
unwinding metadata
debugger support
CET-aware switching
AArch64 support
benchmarking
```

The project does not need all of these to be useful.

They are just natural places to go once the basic mechanism is understood.

---

# Final note

This project is intentionally low-level.

There is no big runtime abstraction hiding the important part.

At the bottom, the mechanism is basically:

```text
stack
+
registers
+
saved rsp
+
return address
+
scheduler
```

The rest is engineering around those pieces.

And that is exactly what this project is for:

**learning what actually happens underneath stackful cooperative execution on x86-64.**

