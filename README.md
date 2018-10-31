# Resource leak detection in OvS-DPDK using gdb's python hooks

This method could also be adapted to find rte_mem leaks (unless that has it's
own leak detection framework) or other resource leaks such as file descriptor
or socket leaks.

An alternative method finding free/malloc leaks using mtrace is linked below
[1].

# Usage

To find leaks in the sample app:


```
  $ build.sh                        # must compile with debug info
  $ sudo gdb a.out                  # start gdb on target program
  gdb> source magic.py              # creates BPs for various resource
                                    # alloc/free fns  also creates new commands
                                    # 'leak start|stop|target|dump'
  gdb> leak start                   # turns on the BPs
  gdb> run
  gdb> leak records leaks.rec       # write alloc/free records to leaks.rec
  gdb> quit
  $ ./parse_leaks.py                # parses leaks.rec and matches malloc/free
                                    # pairs
```

The gdb commands above are already included in the .gdbinit file. And will
run automatically when gdb is started if gdb is configured to do so.

# OvS memory fns

One of the difficulties with OvS's use of malloc/calloc is that calls are
always wrapped in one or more levels of other function calls. These scripts
take account of that and indicate the function that made the alloacation call.

```
	xrealloc ----------------> realloc //straight shim
	xzalloc  --> xcalloc ----> calloc  //calloc (3) zero's alloc'd mem
	                   '-----> malloc  //if size and count are both 0 then mallocs one byte
	                                     //prob to ensure that sz=0 alloc is still freeable?
	xmemdup  --> xmalloc ----> malloc  //alloc and copies from src
	xmemdup0 ----^                     //allocs and zero's extra byte for null-term
```

# Gotchas
```
Python Exception <class 'ValueError'> Variable 'size' not found.:

Breakpoint N, 0x000000.... in FN ()
```

* You probably haven't compiled with debug info. So scripts can't get the
'size' argument to the allocation fn.

# TODOs

* Remove the need to compile with -g.

* Remove the gdb 'chattiness' when creating Finish Breakpoints.

* Also record file and lineno information about the allocation.

## References

[1] https://github.com/sugchand/ovs-dpdk-memleek

[2] https://sourceware.org/gdb/wiki/PythonGdbTutorial

[3] https://sourceware.org/gdb/onlinedocs/gdb/Breakpoints-In-Python.html#Breakpoints-In-Python

[4] https://sourceware.org/gdb/onlinedocs/gdb/Commands-In-Python.html
