# gdb_python_api
Experiments with the GDB Python API

Usage: `PYTHONPATH=/path/to/gdb_python_api gdb ...`

## Backtrace Cleanup for C++ template libraries

Backtraces from heavily templated library code can be tough for users to understand. Through the use of the [frame filter](https://sourceware.org/gdb/onlinedocs/gdb/Frame-Filter-API.html#Frame-Filter-API) and [frame decorator](https://sourceware.org/gdb/onlinedocs/gdb/Frame-Decorator-API.html#Frame-Decorator-API) APIs we can filter out library internals and display more readable aliases for common types.

When the `backtrace` module is imported, future backtraces are controlled by the `backtrace-strip-regex` parameter. Any sequence of frames with matching function names will be trimmed to just the bottom (highest numbered) one. This has the effect of showing only the *call* into library code, and not the subsequent library internals.

In addition, the display of each frame is trimmed by using common type aliases. For example, `std::__cxx11::basic_string<char>` is replaced by `std::string`.

~~~
(gdb) python import gdb_util.backtrace
(gdb) b main
(gdb) run
(gdb) show backtrace-strip-regex
^(std::|__gnu)
step into some function
(gdb) backtrace
~~~

## Stepping only into user code

Another challenge with using template libraries is in stepping through code execution. Particularly in debug builds, such libraries may make a lot of calls that are hard to understand, before reaching any user code. Users can work around this by looking up line numbers and setting breakpoints, but that's tedious.

The `stepu` command steps only into *user* code, by skipping functions matching the `stepu-ignore-regex` parameter. It examines the AST at the current line using [libClang](https://clang.llvm.org/doxygen/group__CINDEX.html)'s Python API, and advances execution until a user function is hit.

The `finishu` command returns you to the point right after where `stepu` was executed, as though you had typed `next` instead.

Due to the use of libClang, an extra environment variable is required:

~~~
PYTHONPATH=/path/to/gdb_python_api LD_LIBRARY_PATH=/usr/lib/llvm-5.0/lib gdb ...
(gdb) python import gdb_util.stepping
(gdb) show stepu-ignore-regex
^(std::|__gnu)
(gdb) b main
(gdb) run
step to a std library function call
(gdb) stepu
~~~

You should now be in the first non-library function called (a plain function, lambda, or object method called by library code).

## Stack frame content display

The command `pframe` gives you a view of the contents of the current (x86) stack frame, showing arguments and local variables in their positions relative to the stack pointer and the beginning of the frame. You can use this to produce an updated display whenever the stack changes by "watching" the stack pointer:

~~~
(gdb) python import gdb_util.stackframe
(gdb) b main
(gdb) run
Breakpoint 1, main (argc=1, argv=0x7fffffffddf8) at...
(gdb) watch $rsp if $_regex($_as_string($rip), ".* <target_fn")
Watchpoint 2: $rsp
(gdb) commands 2
Type commands for breakpoint(s) 2, one per line.
End with a line saying just "end".
>pframe
>end
(gdb) continue
~~~

The stack pointer changes quite frequently so you will probably want to restrict it with a condition like I have above (to `target_fn` and its inlined children).
