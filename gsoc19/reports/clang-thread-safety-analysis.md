# Clang Thread Safety Analysis

It is a static analysis tool which uses annotations to declare and
enforce thread safety policies in C and C++ programs. It warns about
potential race conditions in code. The analysis is not inter-procedural,
so caller requirements must be explicitly declared.

https://clang.llvm.org/docs/ThreadSafetyAnalysis.html

# Running the analysis

To run the analysis, we need to pass `-Wthread-safety` flag:

```
clang -c -Wthread-safety foo.c
```

To run the analysis on the linux kernel, we simply pass the flag
using `CFLAGS_KERNEL`:

```
make CC=clang HOSTCC=clang CFLAGS_KERNEL="-Wthread-safety"
```

# Problems encountered during the investigation

I started annotating the kernel source code in `allnoconfig` to
have fewer warnings and eventually moving to `defconfig`.

1. Lexical scoping: Annotating functions requires the lock instance to
be passed as an argument to the clang annotation. Now this locks instance(variable)
must be defined globally before actually using it to avoid `undeclared identifier`
compiler warning. Such a constraint causes large number of cases to go
unhandled.
https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/28#issuecomment-496215839

Therefore, to silence the bogus warning currently we mark all those with
`__no_thread_safety_analysis`.

2. Name of capability should either be `role` or `mutex`: Linux kernel has many
locking interfaces available which needs to be handled. Analyis gives bogus repeated
warnings in case we decide to use a different name.

For eg.,

```
In file included from /home/himanshu/clang-thread-safety-analysis/kernel/bounds.c:14:
/home/himanshu/clang-thread-safety-analysis/include/linux/spinlock_types.h:73:27: warning: invalid capability name 'spinlock';
      capability name must be 'mutex' or 'role' [-Wthread-safety-attributes]
} spinlock_t __capability("spinlock");
```

3. Annotations can't be applied on macros: The analysis doesn't allow annotating
macros. Unfortunately, linux kernel has lots of those functions wrapped into a
macro. We used a hack to convert those macros into functions but that is not
the best solution to be accepted upstream.
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/33

4. Conditional locking/unlocking: Documentation mentions conditional locking/unlocking
as a limitation. But there are many warnings we certainly can't handle effectively
resulting in silencing it with `__no_thread_safety_analysis`. Silencing makes the analysis
even worse since it turns off analysis for that particular function which might have
handled other warnings when called in a different function.

One peculiar condition of conditional locking/unlocking that analysis fails to
handle is where condition for locking and unlocking are same:
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/140
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/52
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/59

5. No strict typechecking of argument passed to the annotations: The analysis does type
checking of the lock instance argument passed to the annotation. It should always be the
type matching the capability otherwise it immediately issues a mismatch warning.

eg.,
```
/home/himanshu/clang-thread-safety-analysis/include/linux/spinlock.h:336:57: warning: 'acquire_capability' attribute requires arguments
      whose type is annotated with 'capability' attribute; type here is 'spinlock_t *' (aka 'struct spinlock *')
      [-Wthread-safety-attributes]
```

There is a difference between `__acquires_spinlock(foo->bar)` & `__acquires_spinlock(&foo->bar)`

But the problem is we only get the warning in cases where this mismatch is immediately visible
in the translation unit.
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/48

6. There are cases where annotation depends on kernel configuration. Expansion of certain
macros depends upon the chosen kernel config.
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/63

7. There is an API where a generic function is used which does the cleanup including
 the unlocking part. This generic function's prototype is declared as an argument
and subsequently used in the function. There is no way we can make the analysis
understand about such a condition.
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/138

8. Aliasing: It is already mentioned as a limitation of the analysis in documentation. We
found many of them and there is exist no workaround to handle those warnings.
* https://github.com/ClangBuiltLinux/thread-safety-analysis/issues/108
