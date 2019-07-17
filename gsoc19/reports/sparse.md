# Sparse


Sparse is a semantic parser or checker used in the kernel source code
for typechecking, lock checking, address-space violation etc. It is
a simple static analysis tool to find various kinds of problems in
the kernel.

To run sparse on a kernel directory:
```
	$ make C=2 drivers/staging/wlan-ng/
```

To pass check-flags `CF` can be used:
```
	$ make C=2 CF="-D__CHECK_ENDIAN__" drivers/staging/wlan-ng/
```

Default check-flags:
```
	CHECKFLAGS     := -D__linux__ -Dlinux -D__STDC__ -Dunix -D__unix__ \
			  -Wbitwise -Wno-return-void $(CF)
```

## Using sparse for lock checking

The following macros are undefined for gcc and defined during a sparse
run to use the "context" tracking feature of sparse, applied to
locking.  These annotations tell sparse when a lock is held, with
regard to the annotated function's entry and exit.

```
__must_hold - The specified lock is held on function entry and exit.

__acquires - The specified lock is held on function exit, but not entry.

__releases - The specified lock is held on function entry, but not exit.

```

If the function enters and exits without the lock held, acquiring and 
releasing the lock inside the function in a balanced way, no
annotation is needed.  The three annotations above are for cases where
sparse would otherwise report a context imbalance.

According to `man sparse`:

```
-Wcontext
  Warn about potential errors in synchronization or other delimited contexts.

 Sparse  supports several means of designating functions or statements that delimit contexts, such as synchronization.
 Functions with the extended attribute __attribute__((context(expression,in_context,out_context)) require the  context
 expression  (for  instance,  a  lock)  to have the value in_context (a constant nonnegative integer) when called, and
 return with the value out_context (a constant nonnegative integer).  For APIs defined via macros, use  the  statement
 form __context__(expression,in_value,out_value) in the body of the macro.

 With  -Wcontext  Sparse  will  warn when it sees a function change the context without indicating this with a context
 attribute, either by decreasing a context below zero (such as by releasing a lock without acquiring it), or returning
 with  a  changed  context  (such as by acquiring a lock without releasing it).  Sparse will also warn about blocks of
 code which may potentially execute with different contexts.

 Sparse issues these warnings by default.  To turn them off, use -Wno-context.
```

## Statistics

Currently, on v5.2-rc2, we have:

```
$ git grep -w '__acquires' | wc -l
391
$ git grep -w '__releases' | wc -l
438
$ git grep -w '__must_hold' | wc -l
105
```

Running sparse check on kernel(defconfig) gives us:

```
$ make C=2 CF="-Wcontext" 2>&1 >/dev/null | grep -w 'context' | wc -l
772
```

Total `772` warnings for context imbalance needs to be addressed with
suitable annotations.

## Investigation

The best way to test a static analysis tool is to create a simple test
program and run the tool on that program:

```
$ cat -n sparse_context_check.c; sparse sparse_context_check.c 
     1	# define __acquires(x)  __attribute__((context(x,0,1)))
     2	# define __releases(x)  __attribute__((context(x,1,0)))
     3	# define __must_hold(x) __attribute__((context(x,1,1)))
     4	
     5	# define __acquire(x)   __context__(x,1)
     6	# define __release(x)   __context__(x,-1)
     7	
     8	static int lock1, lock2;
     9	
    10	static void lockfn (int i) __acquires(lock1) {
    11	        __acquire(lock1);
    12	        i++;
    13	}
    14	
    15	static void unlockfn (int i) __releases(lock1) {
    16	        __release(lock1);
    17	        i--;
    18	}
    19	
    20	static void unlockfn2 (int i) __releases(lock2) {
    21	        __release(lock2);
    22	        i--;
    23	}
    24	
    25	static void bad_difflocks (int j) {
    26	        lockfn(j);
    27	        unlockfn2(j);
    28	}
    29	
    30	static void bad_onlyunlock (void) {
    31	        int check = 0;
    32	        unlockfn(check);
    33	}
    34	
    35	static void bad_onlylock (void) {
    36	        int check = 0;
    37	        lockfn(check);
    38	}
```

Results:

```
sparse_context_check.c:30:13: warning: context imbalance in 'bad_onlyunlock' - unexpected unlock
sparse_context_check.c:35:13: warning: context imbalance in 'bad_onlylock' - wrong count at exit
```

More test cases can be found at:
https://git.kernel.org/pub/scm/devel/sparse/sparse.git/tree/validation/context.c


## Limitations

* Analysis fails to distinguish between different locks as observed in
 `bad_difflocks` where we expect the tool to warn the about different
  context.

* Even if we omit the argument passed to the annotations, there is no
  effect on the analysis i.e., tool simply doesn't care about the
  lock variable passed as the argument.

## Resources

* https://www.kernel.org/doc/html/latest/dev-tools/sparse.html
* https://lwn.net/Articles/689907/
* https://git.kernel.org/pub/scm/devel/sparse/sparse.git/tree/
* https://lwn.net/Articles/109066/
