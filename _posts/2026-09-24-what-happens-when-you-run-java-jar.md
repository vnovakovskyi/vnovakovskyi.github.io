---
title: "What actually happens when you run java -jar"
date: 2026-09-24
tags: [java, jvm, bytecode, jit]
excerpt: "I went back to the fundamentals after years of writing Java services, and found four things I had quietly wrong: where the standard library lives, what the JVM really is, what the JIT compiles, and which threads keep a service alive."
---

I have been writing Java for years. Usual job, writing services, fixing bugs. Like a normal developer does. But recently I found out that I started to forget basics and things that are living under the hood.
I just imagine what could happen if someone of my colleagues asks me about JVM, JMM, bytecode, and JIT. And having 15+ years of experience, I must answer this correctly.

<!--more-->

So this is the walkthrough I wish someone had given me many years ago: what happens between the code you write and the instructions your CPU executes.

## The chain, end to end

Let's start with the boring part.

<img class="diagram" src="../assets/diagrams/jvm-01-build.svg" alt="One chain: sources go into javac and come out as bytecode, bytecode goes into maven package and comes out as app.jar, app.jar goes into docker build and comes out as an image that already carries a runtime.">

Three points:

1. **javac does almost no optimization.** It checks types and desugars syntax. Beyond folding constant expressions and inlining `static final` constants, it translates your code into bytecode almost one to one. It does not inline methods, it does not unroll loops. JIT will do all of that later, at runtime. This is why bytecode reads so much like the source it came from and why decompilers work so well.

2. **Packaging compiles nothing.** `mvn package` runs the whole lifecycle up to the package phase, compile and tests included. But the packaging step itself only takes the class files that already exist and zips your classes and resources into a jar. Dependencies from `~/.m2` end up inside that jar only if a plugin builds a fat jar, such as `spring-boot-maven-plugin`. A jar is a zip file with a `META-INF` folder in it.

3. **`docker build` knows nothing about Java.** It copies a file into a filesystem layer. Everything Java-specific comes from the base image.


**So what the JDK and the JRE are and what they are holding?**

The standard library is not a build-time thing. Your code calls `java.util` and `java.io` every time it runs, so the library has to be present at runtime. The nesting is:

```
JVM = the machine that executes bytecode
JRE = JVM + the standard class library + the java launcher
JDK = JRE + development tools (javac, jar, javadoc, jdeps, jlink, jshell, jdb)
```

So the difference between a JDK and a JRE is not "libraries versus execution". A JRE is everything you need for *run*. A JDK is that plus everything you need for *build*. The main thing missing from a JRE is `javac`.

And your third-party libraries — Spring, Jackson, a JDBC driver — are in neither. They come from your build, either on the classpath or packed inside the fat jar.

<img class="diagram" src="../assets/diagrams/jvm-01b-class-sources.svg" alt="Three sources of classes at runtime: the standard library ships with the runtime, your own classes and third-party libraries come from your build, and a running JVM loads all three.">

On a server it looks like this:

<img class="diagram" src="../assets/diagrams/jvm-02-run.svg" alt="One chain: the image goes into the orchestrator and comes out as a running container, the container runs the java launcher, and the launcher loads libjvm and becomes a JVM process.">

This is where the runtime physically lives in production: inside the image. A base image like `eclipse-temurin:21-jre` is Linux plus a directory containing `bin/java` and `lib/modules`. Your jar sits on top of it.

You can also build that directory yourself with `jlink`, taking only the modules your app needs:

```bash
jlink --add-modules java.base,java.sql --strip-debug --no-man-pages --no-header-files --output /runtime
```

That is also the answer to a question that confused me: why hasn't Oracle shipped a standalone JRE since Java 11? Because since Java 9 the runtime is a set of modules, and anyone can assemble a runtime that fits their application exactly. Distributions like Temurin still publish JRE builds for convenience. But who needs a universal one?

Also, since Java 10, the JVM reads cgroup limits, so it sees the memory and CPU of the *container*, not of the host. But by default it takes only about a quarter of the available memory for the heap, which is why you usually see `-XX:MaxRAMPercentage=75` in container setups.

## The JVM is just a process


For a long time I was taking JVM as a sandbox for Java code. But the JVM is an ordinary operating system process. It is a program written in C++, it sits on disk as `libjvm.so`, it gets started, it reads files, and allocates memory. Your bytecode is just a *data* that this program reads. Class loaders, the interpreter, the JIT, the garbage collector — none of them are separate entities. They are threads and data structures inside one process.

<img class="diagram" src="../assets/diagrams/jvm-03-process.svg" alt="Inside one java process: class loaders and the execution engine, plus memory areas — heap, metaspace, thread stacks and the code cache. Machine code runs on the CPU.">

"Sandbox" used to mean the security model: the SecurityManager, applets, code you did not trust. That model is gone. It was deprecated for removal in Java 17 and permanently disabled in JDK 24. The JVM is no longer a security boundary: code inside it can read files, open sockets and call native code.
But let's break it down.

A plain `java -jar` process was never sandboxed. The sandbox was opt-in: you got one only when something installed a SecurityManager, and the place that did it by default was the browser — applets, Web Start, code nobody trusted. With a SecurityManager in place, the JDK checked sensitive calls against a policy file: opening a file or connecting a socket asked permission first. With none installed, which is every ordinary server application, those checks did nothing at all. And that mechanism is now permanently disabled, its API is scheduled for removal.

What the JVM does guarantee is memory safety. Bytecode cannot express an address in the first place: fields are read symbolically through the constant pool, array elements through instructions that check the bounds before they touch anything. The verifier proves, before a method ever runs, that it will not pop more than it pushed or use an `int` where a reference belongs. And the GC moves objects around, so an address would be stale a millisecond after you got it.

So the JVM protects the program from itself: you cannot reach memory you were never handed a reference to. The operating system protects the machine from the program: you cannot reach another process. The isolation in production comes from the container, the user account, and the kernel, but not from the JVM. It is an execution environment, not a boundary.

But what lives in the process itself:

- **Heap** — every object you create with `new`. Shared by all threads, managed by the GC. Each loaded class also gets its `java.lang.Class` object here, the *mirror*, and the static fields live right inside it, after the mirror's own fields. A static field of a reference type holds just the reference. The object it points to is an ordinary heap object.
- **Metaspace** — class metadata: the class structure itself (`InstanceKlass` in HotSpot), descriptions of its fields and methods, the bytecode of those methods, and the constant pool. This is where a class loader puts a loaded class. It is native memory, outside the heap. Static field values are not here: `InstanceKlass` keeps only a pointer to the mirror and the offsets of the static fields inside it.
- **Stacks** — one per thread. Frames with local variables, arguments, and return addresses.
- **Code cache** — where the JIT writes machine code.

And here is the startup sequence:

1. The `java` launcher is a small native program. It parses arguments, finds `libjvm` (that is HotSpot) and loads it.
2. Ergonomics reads the limits of the machine or container and picks the heap size and the garbage collector. Memory gets reserved.
3. Service threads start: GC threads, compiler threads, the signal handler.
4. The bootstrap class loader loads the core from `lib/modules` into metaspace: `Object`, `String`, `Class`, `Thread`. That is hundreds of classes, all before your code. To make it fast, the JDK ships a CDS archive — a prepared snapshot of those classes that is simply mapped into memory.
5. The launcher's Java side opens the jar, reads `Main-Class` from `META-INF/MANIFEST.MF`, and the application class loader loads that class.
6. Calling `main` triggers linking (static fields get default values) and initialization (`static {}` blocks run). Only then does the first line of `main` execute.
7. From there, classes are loaded lazily — each one the first time something touches it.

<img class="diagram" src="../assets/diagrams/jvm-03b-startup.svg" alt="Seven startup steps: the launcher loads libjvm, ergonomics picks the heap and GC, service threads start, core classes load, then your main class loads, links, initializes and runs, with every other class loaded lazily.">

That last point explains a failure when a missing class does not blow up at startup. It blows up with `NoClassDefFoundError` at the exact moment execution reaches the code that needs it, possibly hours/days later. It also explains why Spring Boot starts slowly: thousands of classes being loaded, verified, and initialized.

Actually, if your app is a Spring Boot fat jar, `Main-Class` in the manifest is not your class at all. It points at Spring Boot's own `JarLauncher`, and your class is listed under `Start-Class`. A plain JVM cannot load a jar nested inside a jar, so Spring Boot installs its own class loader first.

## Bytecode and the interpreter

Bytecode is an instructions list for a *stack machine*. There are no registers and no memory addresses in it — only "push this onto the stack" and "pop that off the stack".

Every method call creates a frame on the thread's stack, and that frame has two separate areas: an array of local variable slots (for an instance method, slot 0 is always `this`), and the operand stack, the working surface where instructions leave their values. `javac` computes the size of both at compile time and writes them into the class file as `max_locals` and `max_stack`. That is why the JVM knows the exact size of a frame before executing anything, and why the verifier can prove in advance that the stack will not overflow and the types will line up.

Take this method:

```java
int add(int a, int b) {
    int c = a + b;
    return c;
}
```

Run `javap -c` on the compiled class and you get six instructions:

```
0: iload_1
1: iload_2
2: iadd
3: istore_3
4: iload_3
5: ireturn
```

Here is what the first four do:

<img class="diagram" src="../assets/diagrams/jvm-04-operand-stack.svg" alt="iload_1 pushes 3 onto the operand stack, iload_2 pushes 4, iadd pops both and pushes 7, istore_3 pops 7 into a local slot and leaves the stack empty.">

`iload_1` means "take slot 1, push it". `iadd` means "pop two, add them, push the result". `istore_3` means "pop the top, write it into slot 3". After that, `c` holds 7 and the stack is empty again. The same addition in x86 machine code is a single `add` instruction on two registers.

**How the interpreter executes this.** In HotSpot it is a template interpreter, and it works like this:

1. At VM startup, machine code is generated for each of the roughly two hundred opcodes — a template for `iload`, one for `iadd`, one for `invokevirtual`, and so on. This happens once, before your classes are loaded.
2. A dispatch table maps opcode to the address of its template.
3. Executing a method means walking the bytes one at a time. Read an opcode, jump to its template, the template does the work and then reads the next opcode and jumps again. There is no outer loop, the dispatch is baked into the end of every template.
4. The top of the operand stack is kept in a register, so not every step has to touch memory.

<img class="diagram" src="../assets/diagrams/jvm-04b-interpreter.svg" alt="The interpreter reads the current opcode 0x60 from the method bytecode, looks its template address up in the dispatch table, jumps into that template — machine code generated once at VM startup — and then dispatches to the next opcode.">

And this is why interpretation is expensive. Every single bytecode costs a table lookup and an indirect jump, and operands live in frame memory rather than registers. Most importantly, the interpreter sees one instruction at a time, so it cannot possibly notice that `iload_1, iload_2, iadd` is just an addition of two registers. The JIT sees the whole method, and that is where its advantage comes from.

The interesting part is that the interpreter never goes away, even in a long-running service:

- **Startup.** Compiling everything would be far more expensive than running it, and most code executes only a handful of times.
- **Profiling.** Call counts and loop iterations are counted while interpreting. Without that profile the optimizer has nothing to work with.
- **Deoptimization.** When a speculative optimization turns out to be wrong, execution has to land somewhere. It lands in the interpreter: the JVM takes the compiled frame apart and rebuilds an interpreter frame at exactly the right instruction.
- **Cold code and freshly loaded classes.**

## The JIT

1. JIT does not compile Java code — the JVM never sees your source, only bytecode. 
2. JIT compiling happens per method and driven by counters.

<img class="diagram" src="../assets/diagrams/jvm-05-jit.svg" alt="Six steps: the method runs in the interpreter while counters tick up, a counter crosses its threshold, a compiler thread compiles it in the background, the machine code lands in the code cache, the entry pointer is updated, and the CPU executes it directly.">

Every method has two counters: how many times it was called, and how many backward jumps its loops made. When a counter crosses a threshold, the method is queued for compilation. A background compiler thread takes its bytecode and produces machine code, while the application keeps running on the interpreter — there is no pause. The result is written into the code cache, a region of native memory inside the same process marked as executable. Then the method's entry pointer is updated, and the next call jumps straight into it.

From that moment the CPU executes those bytes directly. Nothing sits in between: not the JVM, not the OS. The JVM was involved exactly once, when it asked the OS for memory it was allowed to execute. It is the same thing a compiled C program does, except the code appeared in memory while the program was running instead of being read from a file.

The speedup does not come from the translation itself. It comes from what becomes possible once you see a whole method: local variables move into CPU registers, small methods get inlined into their callers, dead branches disappear, loops get unrolled. On hot code the difference against the interpreter is tens of times.

A few points I want to explicitly mention:

**Compilation is tiered.** Level 0 is the interpreter. Level 3 is C1: compiles quickly, produces mediocre code, but collects a profile — which branches are taken, which types actually show up at a call site. Level 4 is C2: slow to compile, but it uses that profile and produces very good code.

**Those optimizations are speculative, and they can be undone.** If C2 observed that only one type ever arrives at a call site, it will inline the call and guard it with a check. When another type shows up, the guard fails, the compiled code is thrown away, and execution continues in the interpreter. That is deoptimization, and it is also why a benchmark without a warmup phase lies to you.

<img class="diagram" src="../assets/diagrams/jvm-05b-tiers.svg" alt="A method moves from level 0 in the interpreter to level 3 where C1 compiles it and collects a profile, then to level 4 where C2 uses that profile. When a speculative guard fails, deoptimization takes execution back to the interpreter.">

There is another case: a method called exactly once, with a loop of a million iterations inside. The call counter will never make it hot, which is what the second counter is for. When it trips, the JVM compiles the method and swaps execution over *in the middle of the running loop*, rebuilding the interpreter frame as a compiled frame. That is on-stack replacement.

<img class="diagram" src="../assets/diagrams/jvm-05c-osr.svg" alt="A method entered once never trips the call counter, but its loop trips the backedge counter. A compiler thread compiles the method while the loop is still running, and on-stack replacement turns the interpreter frame into a compiled frame between two iterations.">

## Five levels, and the routes between them

A method is never simply "interpreted" or "compiled". At any moment it sits at one of five execution levels, and the level says two things: who produced the code that is running, and how much bookkeeping that code carries.

| Level | Runs the code | Collects |
| ----- | ------------- | -------- |
| 0 | the interpreter | call and loop counters |
| 1 | C1 | nothing |
| 2 | C1 | counters only |
| 3 | C1 | a full profile: branches taken, types seen at each call site |
| 4 | C2 | nothing |

The numbers are not a scale of optimization. Levels 1, 2 and 3 are the same compiler producing the same quality of code. What differs is the instrumentation baked into it. Sort them by speed instead and the order comes out as 0 ≪ 3 < 2 < 1 < 4 — level 1 code is faster than level 3 code, because it never stops to record anything.

That is the whole logic of the ladder. Profiling is not free, so the JVM pays for it only where the data will actually be read.

<img class="diagram" src="../assets/diagrams/jvm-06-tier-routes.svg" alt="Five routes between levels: the usual path 0 to 3 to 4, a trivial method 0 to 1, a long C2 queue 0 to 2 to 3 to 4, a long C1 queue 0 straight to 4, and deoptimization from 4 back to 0.">

**0 → 3 → 4** is the common route. The counters trip in the interpreter, C1 compiles the method with full profiling, that compiled code keeps counting, and when it stays hot C2 compiles it again using the profile.

**0 → 1** happens when the method is trivial — an accessor or a method that returns a constant. The policy recognises it before any profiling and sends it straight to C1 without instrumentation: C2 would produce the same code, so a profile would be paid for and never read.

**0 → 2 → 3 → 4** is what a long C2 queue looks like. Running slow profiled code for the whole wait would be wasteful, so the method first gets fast C1 code with cheap counters, and the full profile is collected later, closer to the real compile.

**0 → 4** is the mirror case: C1 is busy, so the profile is gathered in the interpreter and the method goes straight to C2.

Two separate decisions hide in those arrows. Whether a method is *worth C2's time* is decided at level 3, by its own counters against the level 4 thresholds — the method is already compiled when that happens, it just stayed hot. Whether C2 can actually do better is decided inside the compiler: it reads the profile and can bail out. Separately, the policy never compiles a method with more than 8000 bytes of bytecode, at any tier (`-XX:-DontCompileHugeMethods` lifts this). The triviality check that sends a method to level 1 is made by the policy before C2 ever sees it.

Deoptimization drops a method back to level 0, where it starts collecting a profile again and may well be recompiled. If it keeps failing, the JVM gives up in three steps rather than all at once. First it drops the single speculation: when the same guard at the same bytecode has failed a handful of times, the method is recompiled without that assumption, so a call site that has now seen three different types becomes a plain virtual call instead of an inlined one. Then it stops speculating in that method at all. Only after a few hundred recompilations is the method marked as not compilable for C2, and it runs as plain C1-compiled code at level 1 for the rest of the process — which in practice means either pathological code or a compiler bug.

There is one more way compilation stops: the code cache fills up. The log says `CodeCache is full. Compiler has been disabled`, and new compilations stop. Code that is already compiled keeps running, but anything that gets hot from now on stays in the interpreter. With the default `-XX:+UseCodeCacheFlushing` the JVM restarts the compiler once enough code is unloaded. It is a real production incident, and it looks like a service that is getting slower for no visible reason.

## Threads decide when the process dies

Why doesn't the Spring Boot process exit when `main` returns? Isn't returning from `main` shuts down the other threads?

The actual rule is:

**The JVM stays alive as long as at least one non-daemon thread is alive.**

When `main` returns, its thread finishes and the JVM checks whether any non-daemon threads are left. If there are, the process keeps running. Daemon threads are the opposite: they hold nothing. Once the last non-daemon thread is gone, the JVM runs the shutdown hooks and kills daemon threads wherever they happen to be — their `finally` blocks do not run.

So for a web service to stay up, it needs a non-daemon thread. Spring Boot's `TomcatWebServer` starts one explicitly, named `container-0`, and calls `setDaemon(false)` on it. It just stands there and waits, holding the JVM alive. Make it a daemon and your application would start, print "Started Application in 3.2 seconds" and immediately exit.

The JVM is not magic and it is not a sandbox. It is a C++ program that reads your bytecode, keeps its own memory in order, and rewrites the hot parts into machine code while your service is running. Everything above is that one sentence, unpacked.

---

> 🤖 The thinking and the words here are mine. AI helped me tighten the language and shape the structure.
>
> If you quote or repost this, please credit the source.
>
> I write about the JVM, reliability and everyday engineering on [LinkedIn](https://www.linkedin.com/in/vadym-novakovskyi/) — follow along if that is your kind of thing.
