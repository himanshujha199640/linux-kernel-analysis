# Smatch

Smatch is static analysis tool built on top of sparse library to
have a data flow analysis to detect how values change across a
sequence of code. Unlike sparse it understands the [semantics](https://repo.or.cz/smatch.git/blob/HEAD:/check_locking.c#l104)
of every lock present in the linux kernel. Therefore, there is no
requirement of annotating the functions in the kernel source. To
make the analysis more accurate it takes advantage from a cross
functional database which has various details of functions and global
variables. Building the database is time consuming but it is a
useful thing to do.

```
        cd ~/path/to/kernel_dir
        ~/path/to/smatch_dir/smatch_scripts/build_kernel_data.sh

```

Rebuilding the cross function database makes it more accurate.

## Using Smatch

If you are running Smatch over the whole kernel you can use the following
command:
```
        ~/progs/smatch/devel/smatch_scripts/test_kernel.sh
```

The `test_kernel.sh` script will create a .c.smatch file for every file it tests
and a combined `smatch_warns.txt` file with all the warnings.

If you are running Smatch just over one kernel file:
```
        ~/progs/smatch/devel/smatch_scripts/kchecker drivers/whatever/file.c
```
You can also build a directory like this:
```
        ~/progs/smatch/devel/smatch_scripts/kchecker drivers/whatever/
```

The kchecker script prints its warnings to stdout.

## Investigation

```
$ cat -n smatch_context_check.c 
     1	void _spin_lock(int name);
     2	void _spin_unlock(int name);
     3	void _spin_unlock_bh(int name);
     4	
     5	extern int foo;
     6	
     7	static void bad_difflocks (int j) {
     8		_spin_lock(j);
     9		_spin_unlock_bh(j);
    10		return;
    11	}
    12	
    13	static void bad_onlyunlock (void) {
    14		int check = 0;
    15		_spin_unlock(check);
    16	
    17		return;
    18	}
    19	
    20	static void bad_onlylock (void) {
    21		int check = 0;
    22		_spin_lock(check);
    23	}
    24	
    25	static void bad_if (int i) {
    26		if(foo)
    27			_spin_lock(j);
    28	
    29		return;
    30	}
```

Result:

```
$ ./smatch -p=kernel --spammy smatch_context_check.c
smatch_context_check.c:29 bad_if() warn: 'spin_lock:j' is sometimes locked here and sometimes unlocked.
```

## Limitation

* Unable to differentiate between locks as observed in `bad_difflocks` above.
* Doesn't warn about cases where only either locking or unlocking takes place.

## Resources

* https://repo.or.cz/smatch.git/blob/HEAD:/check_locking.c
* https://lwn.net/Articles/691882/
