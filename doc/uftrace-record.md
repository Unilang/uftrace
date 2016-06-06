% UFTRACE-RECORD(1) Uftrace User Manuals
% Namhyung Kim <namhyung@gmail.com>
% May, 2016

NAME
====
uftrace-record - Run a command and record its trace data

SYNOPSIS
========
uftrace record [*options*] COMMAND [*command-options*]

DESCRIPTION
===========
This command runs COMMAND and gathers function trace from it, saves it into files under uftrace data directory - without displaying anything.

This data can then be inspected later on, using `uftrace-replay` or `uftrace-report` command.

OPTIONS
=======
-b *SIZE*, \--buffer=*SIZE*
:   Size of internal buffer which trace data will be saved.  Default is 128k.

-F *FUNC*, \--filter=*FUNC*
:   Set filter to trace selected functions only.  This option can be used more than once.  See *FILTERS*.

-N *FUNC*, \--notrace=*FUNC*
:   Set filter not trace selected functions only.  This option can be used more than once.  See *FILTERS*.

-T *TRG*, \--trigger=*TRG*
:   Set trigger on selected functions.  This option can be used more than once.  See *TRIGGERS*.

-t *TIME*, \--time-filter=*TIME*
:   Do not show small functions under the time threshold.  If some functions explicitly have 'trace' trigger, those are always traced regardless of execution time.

\--force
:   Allow to run uftrace even if some problem might occur.  When `uftrace-record` finds no mcount symbol (which is generated by compiler) in the executable it quits with an error message since it there's no need to run the program.  However it's possible one is only interested functions in a library, in this case she can use this option so uftrace can keep running the program.  Also -A,\--argument and -R,\--retval option works only for binaries built with -pg so it makes uftrace exit when it tries to run other binaries.  This option ignores the warning and go tracing without the argument and/or return value.

-L *PATH*, \--library-path=*PATH*
:   Load necessary internal libraries from this path.  This is for testing purpose.

\--no-plthook
:   Do not record library function invocations.  The uftrace traces library calls by hooking dynamic linker's resolve function in the PLT.  One can disable it with this option.

\--no-pltbind
:   Do not bind dynamic symbol address.  This option uses the LD_BIND_NOT environment variable to trace library function calls which might be missing due to concurrent (first) accesses.  It's only meaningful to use this option without the --no-plthook option.

-D *DEPTH*, \--depth=*DEPTH*
:   Set global trace limit in nesting level.

\--max-stack=*DEPTH*
:   Set max stack depth to trace function.  Default is 1024.

\--nop
:   Do not record any functions.  It's a no-op and only meaningful for performance comparison.

\--time
:   Print running time of children in time(1)-style.

-k, \--kernel
:   Trace kernel functions as well as user functions.  Only kernel entry/exit functions will be traced as if -D 1 was given.

-K, \--kernel-full
:   Trace kernel functions as well as user functions.  Kernel functions will be traced in full detail (ie. without depth limit unless explicitly given)

-H *HOST*, \--host=*HOST*
:   Send trace data to given host via network, not writing to files.  The `uftrace-recv` should be run on the host to receive the data.

\--port=*PORT*
:   When sending data to network (with -H option), use given port instead of the default port (8090).

\--disable
:   Start uftrace with tracing disabled.  This is only meaningful when used with 'trace_on' trigger.

\--demangle=*TYPE*
:   Use demangled C++ symbol names for filters, triggers, arguments and/or return values.  Possible values are "full", "simple" and "no".  Default is "simple" which ignores function arguments and template parameters.

-A *SPEC*, \--argument=*SPEC*
:   Record function arguments.  This option can be used more than once.  See *ARGUMENTS*.

-R *SPEC*, \--retval=*SPEC*
:   Record function return value.  This option can be used more than once.  See *ARGUMENTS*.

\--num-thread=*NUM*
:   Use NUM of thread to record trace data.  Default is 1/4 of online cpus (but when full kernel tracing is enabled, it'll use the same number of the cpus).


FILTERS
=======
The uftrace support filtering only interested functions.  Filtering is highly recommended since it helps users focusing on the interested functions and reduces the data size.  When uftrace is called it receives two types of function filter; opt-in filter with -F/\--filter option and opt-out filter with -N/\--notrace option.  These filters can be applied either record time or replay time.

The first one is an opt-in filter; By default, it doesn't trace anything and when it executes one of given functions it starts tracing.  Also when it returns from the given funciton, it stops again tracing.

For example, suppose a simple program which calls a(), b() and c() in turn.

    $ cat abc.c
    void c(void) {
        /* do nothing */
    }

    void b(void) {
        c();
    }

    void a(void) {
        b();
    }

    int main(void) {
        a();
        return 0;
    }

    $ gcc -pg -o abc abc.c

Normally uftrace will trace all the functions from `main()` to `c()`.

    $ uftrace ./abc
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

But when `-F b` filter option is used, it'll not trace `main()` and `a()` but only `b()` and `c()`.

    $ uftrace record -F b ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
                [ 1234] | b() {
       3.880 us [ 1234] |   c();
       5.475 us [ 1234] | } /* b */

The second type is an opt-out filter; By default, it trace everything and when it executes one of given functions it stops tracing.  Also when it returns from the given funciton, it starts tracing again.

In the above example, you can omit the function b() and its children with -N option.

    $ uftrace record -N b ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
       6.448 us [ 1234] |   a();
       8.631 us [ 1234] | } /* main */

In addition, you can limit the print nesting level with -D option.

    $ uftrace record -D 3 ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

In the above example, it prints functions up to 3 depth, so leaf function c() was omitted.  Note that the -D option works with -F option.

Sometimes it's useful to see long-running functions only.  This is good because there're many tiny functions that are not interested usually.  The -t/\--time-filter option implements the time-based filter that only records functions run longer than the given threshold.  In the above example, user might want to see functions running more than 5 micro-seconds like below:

    $ uftrace record -t 5us ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

The -t/\--time-filter option works for user-level functions only.

You can also set triggers to filtered functions.  See *TRIGGERS* section below for details.

When kernel function tracing is enabled, you can also set the filters on kernel functions.  It needs to mark the symbol with '@kernel' modifier.  Following example will show all user functions and (kernel) page fault handler.

    $ sudo uftrace -k -F '*page_fault@kernel' ./abc
    # DURATION    TID     FUNCTION
               [14721] | main() {
      7.713 us [14721] |   __do_page_fault();
      6.600 us [14721] |   __do_page_fault();
      6.544 us [14721] |   __do_page_fault();
               [14721] |   a() {
               [14721] |     b() {
               [14721] |       c() {
      0.860 us [14721] |         getpid();
      2.346 us [14721] |       } /* c */
      2.956 us [14721] |     } /* b */
      3.340 us [14721] |   } /* a */
     79.086 us [14721] | } /* main */


TRIGGERS
========
The uftrace supports triggering some actions on selected function with or without filters.  Currently supported triggers are listed below.  The BNF for the trigger is:

    <trigger>  :=  <symbol> "@" <actions>
    <actions>  :=  <action>  | <action> "," <actions>
    <action>   :=  "depth=" <num> | "trace" | "trace_on" | "trace_off" | "recover"

The depth trigger is to change filter depth during execution of the function.  It can be use to apply different filter depths for different functions.  And the backrace trigger is to print stack backtrace at replay time.

Following example shows how trigger works.  The global filter depth is 5, but function 'b' changed it to 1 so functions below the 'b' will not shown.

    $ uftrace record -D 5 -T 'b@depth=1' ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main() {
                [ 1234] |   a() {
       5.475 us [ 1234] |     b();
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } /* main */

The 'backtrace' trigger is only meaningful in replay command.  The 'traceon' and 'traceoff' (you can omit '_' between 'trace' and 'on/off') controls whether uftrace records functions or not.

The 'recover' trigger is for some corner cases which the process accesses the callstack directly.  During tracing the v8 javascript engine, it kept get segfault in the garbage collection stage.  It was because the v8 interpretes the return address into compiled code object(?).  The 'recover' trigger restores the original return address at the function entry and reset to the uftrace's return hooking address again at the function exit.  I was managed to work around the segfault by setting 'recover' trigger on the related function (specifically ExitFrame::Iterate).

The uftrace trigger only works for user-level functions for now.


ARGUMENTS
=========
The uftrace supports recording function arguments and/or return value using -A/\--argument and -R,\--retval options respectively.  The syntax is very similar to the trigger:

    <argument> :=  <symbol> "@" <specs>
    <specs>    :=  <spec>  | <spec> "," <spec>
    <spec>     :=  ( "arg" N | "retval" ) [ "/" <format> [ <size> ] ]
    <format>   :=  "i" | "u" | "x" | "s" | "c"
    <size>     :=  "8" | "16" | "32" | "64"

The -A,\--argument option takes argN where N is an index of the arguments.  The index starts from 1 and corresponds to argument passing order of the calling convention on the system.  Currently it assumes all the arguments are integer (or pointer) type, so the result might not correct if there're floating-point arguments as well.  The -R,\--retval option takes "retval" and also assumes it's an integer type.

Users can specify optional format and size of the arguments and/or return value.  Without this, the uftrace treats them as 'long int' type and print them appropriately.  The "i" format makes it signed integer type and "u" format is for unsigned type.  Both are printed as decimal while "x" format makes it printed as hexa-decimal.  The "s" format is for string type and "c" format is for character type.

    $ uftrace record -A main@arg1/x -R main@retval/i32 ./abc
    $ uftrace replay
    # DURATION    TID     FUNCTION
     138.494 us [ 1234] | __cxa_atexit();
                [ 1234] | main(0x1) {
                [ 1234] |   a() {
                [ 1234] |     b() {
       3.880 us [ 1234] |       c();
       5.475 us [ 1234] |     } /* b */
       6.448 us [ 1234] |   } /* a */
       8.631 us [ 1234] | } = 0; /* main */

Note that these arguments and return value are recorded only if the executable was built with -pg option.  The executables built with -finstrument-functions will exit with an error message.  Also it only works for user-level functions for now.


SEE ALSO
========
`uftrace`(1), `uftrace-replay`(1), `uftrace-report`(1), `uftrace-recv`(1)