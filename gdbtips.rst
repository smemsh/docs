GDB Usage Tips
==============================================================================

Collection of accumulated tips for debugging with GDB.


Conditional breakpoints
-----------------------

Breakpoint conditions can be added with an ``if`` and can be expressions in the
language being debugged.  But there are "convenience" functions implemented in
Python such as:

- set a breakpoint that gets called from a function named
  ``Variant::cast`` (C++) anywhere within 3 stack layers with the
  ``input`` parameter set to the string "2022-W52-1"::

    (gdb) break src/libshared/src/Datetime.cpp:186 if ( \
      $_any_caller_matches("Variant::cast") && \
      (input.compare("2022-W52-1") == 0) \
    )

- for the string compare, it can be done internally withing
  calling into program-linked libraries eg ``$_streq(x, "hello")``

Other convenience functions are described at:
https://sourceware.org/gdb/current/onlinedocs/gdb.html/Convenience-Funs.html

Breakpoints described more completely at:
https://sourceware.org/gdb/current/onlinedocs/gdb.html/Set-Breaks.html

also see ``/usr/share/gdb/python/gdb/**/*.py``
