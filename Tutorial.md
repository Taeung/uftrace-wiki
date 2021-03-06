#basic usage
The uftrace tool consists of several subcommands.  We'll see how to use them with simple examples in this document. Note that this document is still work in progress and based on behavior of the latest version of uftrace.

## Getting started
The first subcommand to look at is the `live`.  It's a default subcommand and will be used if you don't give other subcommand when running uftrace (So it's same to run "uftrace xxx" and "uftrace live xxx").  It's basically same as running `record` and then `replay` subcommands in a row.  That means it'd show the output after a program (given on the command line) finished.  Below is the familiar "hello world" program.

    $ cat hello.c
    #include <stdio.h>
    
    int main(void) {
      printf("Hello world\n");
      return 0;
    }

To analyze this program with uftrace, you need to enable the compiler instrumentation.  The gcc provides `-pg` and `-finstrument-functions` options for this.  We prefer to use `-pg` option as it's more light-weight in terms of compiler optimization, but there were some cases it didn't work well.  Currently function arguments are only accessible if it's compiled with the `-pg` option.

    $ gcc -o hello -pg hello.c

Now you can run the hello program with uftrace like below:

    $ uftrace hello
    Hello world
    # DURATION    TID     FUNCTION
       1.337 us [26639] | __monstartup();
       0.897 us [26639] | __cxa_atexit();
                [26639] | main() {
       6.696 us [26639] |   puts();
       7.582 us [26639] | } /* main */

The first line is, as we expect, the output from the hello program.  The following lines are from the uftrace and shows function execution time, process/thread id and function names.  As with function graph tracer in the Linux kernel, the functions are shown as a source code of C programming language.  You can see that the "puts" functions was called during "main" instead of "printf".  It's an optimization of the compiler (gcc) to replace it when the format string has no conversion specifier (e.g. %d) and ends with a new line character.

Note that it showed the "puts" which is a library function you didn't write (the same goes to "__monstartup" and "__cxa_atexit") as well as your own functions in the program.  Of course it doesn't show the internals of the "puts" but it's good to see how long does the "puts" take.  It uses a technique called "PLT hooking" which redirects functions called from a dynamically-linked program.  While it's very powerful (so it's enabled by default), it comes with its overhead too.  So if you don't want to see them and/or don't want to take the overhead, you can use `--no-libcall` option for `uftrace record` to disable it.  As `live` subcommand can take all options the `record` takes, you can run like this:

    $ uftrace --no-libcall hello
    Hello world
    # DURATION    TID     FUNCTION
       7.534 us [26714] | main();

Now you can see only the "main" function - you wrote it only, right? :)   The "Hello world" in the output shows that it actually calls the "puts" function but it didn't show up in the output from uftrace.  Also, you might notice that the "main" function is shown on a single line now.  This is because uftrace shows leaf functions which don't call other functions (from the uftrace's perspective) in the compact format.

## Recording trace data
The uftrace needs to collect trace data in order to analyze the program execution.  Many subcommand in uftrace requires the data to run.  The `record` subcommand saves the trace data in the "uftrace.data" directory by default, and other subcommands use it.  You can use `-d` or `--data` option to use different name.

Note that you should give uftrace options before the (path of) your program.  The uftrace stops processing command line options when it sees a non-option argument and passes the rest to the program as is.

As you already know, you can give the program name on the command line.

    $ uftrace record pwd

However it'll show the following error message and exit.

    uftrace: /home/namhyung/project/uftrace/cmd-record.c:1271:check_binary
     ERROR: Cannot trace 'pwd': No such file
            Note that ftrace doesn't search $PATH for you.
            If you really want to trace executables in the $PATH,
            please give it the absolute pathname (like /usr/bin/pwd).

This is because it doesn't search directories in the PATH environment variable for you.  In order to run with uftrace, the program needs to be built with the compiler instrumentation.  But programs in the PATH are usually not.  So you need to give the full path to run if you really want to run it with the uftrace.  Let's do this:

    $ uftrace record `which pwd`
    uftrace: /home/namhyung/project/uftrace/cmd-record.c:1307:check_binary
     ERROR: Can't find 'mcount' symbol in the '/usr/bin/pwd'.
            It seems not to be compiled with -pg or -finstrument-functions flag
            which generates traceable code.  Please check your binary file.

It still shows the error message and exits.  This is because the program (pwd) was not built with the compiler instrumentation so there's nothing uftrace can trace.  However you might remember that it can also trace library functions.  If you want to trace calls to library functions even though the program itself was not built to be traced, you can use `--force` option:

    $ uftrace record --force `which pwd`
    /home/namhyung/project/uftrace

The output would be a series of library calls which look like a very simple version of ltrace.  You can use `replay` subcommand to see the recorded program execution.

    $ uftrace replay
    # DURATION    TID     FUNCTION
       1.716 us [26891] | getenv();
       0.994 us [26891] | strrchr();
      60.438 us [26891] | setlocale();
       2.244 us [26891] | bindtextdomain();
       1.152 us [26891] | textdomain();
       0.792 us [26891] | __cxa_atexit();
       1.449 us [26891] | getopt_long();
       4.977 us [26891] | getcwd();
      15.407 us [26891] | puts();
       1.249 us [26891] | free();
       0.985 us [26891] | __fpending();
       0.786 us [26891] | fileno();
       0.800 us [26891] | __freading();
       0.223 us [26891] | __freading();
       3.897 us [26891] | fflush();
       2.635 us [26891] | fclose();
       0.180 us [26891] | __fpending();
       0.166 us [26891] | fileno();
       0.166 us [26891] | __freading();
       0.137 us [26891] | __freading();
       0.283 us [26891] | fflush();
       0.617 us [26891] | fclose();

Basically uftrace will save trace of every single function call (and return).  But it's huge and sometimes impossible to do it for long-running and/or heavy-weight programs.  So uftrace provides a couple of filtering options to control the trace data.  Although some of the filtering can work at later processing (like replay), it'd be better to reduce the amount of data at record time.

One is function-level filters and works on the name of functions.  You can use `-F` or `--filter` option to specify a function to trace.  With this option, all functions called during the function will be recorded and *NO* functions called outside of the function will be recorded.  Also there're `-N` or `--notrace` option to do it in an opposite way.  All functions called during the function will *NOT* be recorded, and all functions called outside of the function will be recorded.  You can also use these options together and more than once.  In the hello world example, you can use it to see function called under "main" only.

    $ uftrace -F main hello
    Hello world
    # DURATION    TID     FUNCTION
                [27132] | main() {
       6.522 us [27132] |   puts();
       8.744 us [27132] | } /* main */

The next is function-depth filter.  You can use `-D` or `--depth` to limit function call depth to be recorded (or replayed).  Below example applies depth-1 filter which shows functions only called in a top level:

    $ uftrace -D 1 hello
    # DURATION    TID     FUNCTION
       1.630 us [27091] | __monstartup();
       0.917 us [27091] | __cxa_atexit();
       7.493 us [27091] | main();

The last type of filter is a time-based one.  It will show functions running longer than the specified time.  Usually short-running functions are out of interest when analyzing program execution so it's useful to remove those function at once.  You can use `-t` or `--time-filter` option like below:

    $ uftrace -t 5us hello
    # DURATION    TID     FUNCTION
                [27154] | main() {
       5.027 us [27154] |   puts();
       6.280 us [27154] | } /* main */

Above shows functions run longer than 5 micro-second (us).  Note that the output when using time filter can be vary for each run due to the various timing issues.  You can also give other units like ns, ms, s or m for nano-second, milli-second, second and minute respectively.  If you omit the unit, it defaults to the "ns".  Please do not put a whitespace between number and unit.

In addition, it can also access function arguments and return value.  You can use the `-A` or `--argument` option to access the arguments and likewise, `-R` or `--return` option for return value.  (Currently) it needs to pass function name and argument/return value specifier(s).

    $ uftrace -A puts@arg1/s -R main@retval hello
    Hello world
    # DURATION    TID     FUNCTION
       2.619 us [27961] | __monstartup();
       2.014 us [27961] | __cxa_atexit();
                [27961] | main() {
       8.256 us [27961] |   puts("Hello world");
       9.996 us [27961] | } = 0; /* main */

The first argument of "puts" function is the string so it needs to add "/s" format specifier at the end.  By default integer type is assumed so retval has no format specifier.  For more information please refer the manual page.

## Replaying the trace
Once you recorded the trace data of your program, you can use it to see the execution of program.  As uftrace saved all the information to replay the trace, it's also easy to do it on a different machine by simply copying the data directory (uftrace.data) and running `uftrace replay` on it.  Most of filtering (except time filter) also works for replay.

Let's take a look at the following silly program.

    $ cat foobar.c
    #include <pthread.h>
    
    void *bar(void) {
      return NULL;
    }
    
    void *foo(void *unused) {
      return bar();
    }
    
    int main(int argc, char *argv[]) {
      pthread_t th;
    
      foo(argv);
      pthread_create(&th, NULL, foo, NULL);
      pthread_join(th, NULL);
      return 0;
    }

It creates a thread and calls foo (and bar) function from the two thread each.  Compile and record the program like below:

    $ gcc -o foobar -pg -pthread foobar.c
    $ uftrace record foobar

Now `replay` shows the execution of program.  Note that it uses a pager program (usually "less" - use can set "PAGER" environment variable to change) to control the terminal output easily.  If you don't want it for some reason, you might use `--no-pager` option or set the "PAGER" env. to "cat".

    $ uftrace replay
    # DURATION    TID     FUNCTION
       2.217 us [22071] | __monstartup();
       2.274 us [22071] | __cxa_atexit();
                [22071] | main() {
                [22071] |   foo() {
       0.234 us [22071] |     bar();
       1.264 us [22071] |   } /* foo */
      68.900 us [22071] |   pthread_create();
                [22071] |   pthread_join() {
                [22073] | foo() {
       0.241 us [22073] |   bar();
       1.819 us [22073] | } /* foo */
     204.783 us [22071] |   } /* pthread_join */
     278.739 us [22071] | } /* main */

You can see only specific thread(s) by using `--tid` option.

    $ uftrace replay --tid 22073
    # DURATION    TID     FUNCTION
                [22073] | foo() {
       0.241 us [22073] |   bar();
       1.819 us [22073] | } /* foo */

There's also `--column-view` option to make it easy to distinguish different threads/processes like below:

    $ uftrace replay --column-view
    # DURATION    TID     FUNCTION
       2.217 us [22071] | __monstartup();
       2.274 us [22071] | __cxa_atexit();
                [22071] | main() {
                [22071] |   foo() {
       0.234 us [22071] |     bar();
       1.264 us [22071] |   } /* foo */
      68.900 us [22071] |   pthread_create();
                [22071] |   pthread_join() {
                [22073] |                 foo() {
       0.241 us [22073] |                   bar();
       1.819 us [22073] |                 } /* foo */
     204.783 us [22071] |   } /* pthread_join */
     278.739 us [22071] | } /* main */

## Analyzing the trace
With the trace data you recorded above, uftrace provides a couple of subcommands to analyze the program execution.  The `report` subcommand shows statistics of each function in the program sorted by (total) time executed.

    $ uftrace report
      Total time   Self time       Calls  Function
      ==========  ==========  ==========  ====================================
      278.739 us    3.792 us           1  main
      204.783 us  204.783 us           1  pthread_join
       68.900 us   68.900 us           1  pthread_create
        3.083 us    2.608 us           2  foo
        2.274 us    2.274 us           1  __cxa_atexit
        2.217 us    2.217 us           1  __monstartup
        0.475 us    0.475 us           2  bar

The total time is a duration of the function including (child) functions called in it.  The self time is a duration excluding the child functions.  If the function is called multiple times, it's a sum of duration.  You can use `-s` or `--sort` option to change the sorting behavior.  Available sort keys are: "total" (default), "self" and "call".  Below is the same trace but sorted by the self time.

    $ uftrace report -s self
      Total time   Self time       Calls  Function
      ==========  ==========  ==========  ====================================
      204.783 us  204.783 us           1  pthread_join
       68.900 us   68.900 us           1  pthread_create
      278.739 us    3.792 us           1  main
        3.083 us    2.608 us           2  foo
        2.274 us    2.274 us           1  __cxa_atexit
        2.217 us    2.217 us           1  __monstartup
        0.475 us    0.475 us           2  bar

Sometimes functions are called more than once and you want to know the average execution time of the function.  In the above example, the "foo" and "bar" functions are called twice - once from the main thread another from the newly created thread.  For this, uftrace provides `--avg-total` and `--avg-self` option to see average of the total or self time.

    $ uftrace report --avg-total
       Avg total   Min total   Max total  Function
      ==========  ==========  ==========  ====================================
      278.739 us  278.739 us  278.739 us  main
      204.783 us  204.783 us  204.783 us  pthread_join
       68.900 us   68.900 us   68.900 us  pthread_create
        2.274 us    2.274 us    2.274 us  __cxa_atexit
        2.217 us    2.217 us    2.217 us  __monstartup
        1.541 us    1.264 us    1.819 us  foo
        0.237 us    0.234 us    0.241 us  bar

In this average mode, the available sort keys are different: "min", "max" and "avg" (default).

Note that the `live` subcommand also has a `--report` option to show this style of output before the replay.

Sometimes you might want to focus on a single function, like how it was called (backtrace) and what functions it calls (children).  The `graph` command shows you such information.  For example, let's see function call graph of the "foo":

    $ uftrace graph foo
    #
    # function graph for 'foo' (session: beb2ed6a788e3fc6)
    #
    
    backtrace
    ================================
     backtrace #0: hit 1, time   1.264 us
       [0] main (0x40073b)
       [1] foo (0x40071c)
    
     backtrace #1: hit 1, time   1.819 us
       [0] foo (0x40071c)
    
    calling functions
    ================================
       3.083 us : (2) foo
       0.475 us : (2) bar

It first shows the backtrace of "foo", one is from "main" and the other is called directly (from the new thread).  Also it shows that the "foo" is called twice and then it also calls "bar" twice.  Below is a graph of "main" function:

    $ uftrace graph
    #
    # function graph for 'main' (session: beb2ed6a788e3fc6)
    #
    
    backtrace
    ================================
     backtrace #0: hit 1, time 278.739 us
       [0] main (0x40073b)
    
    calling functions
    ================================
     278.739 us : (1) main
       1.264 us :  +-(1) foo
       0.234 us :  | (1) bar
                :  | 
      68.900 us :  +-(1) pthread_create
                :  | 
     204.783 us :  +-(1) pthread_join

As you can see, if you omit the command line argument (function name), it'll show "main" function by default.  It has a single backtrace and called 3 functions - "foo", "pthread_create" and "pthread_join".  The "bar" was called from "foo" (which is called from "main" of course) in this case.  As it only focuses on the "main", "foo" (and "bar") called from the different thread wasn't shown here.

Finally, uftrace can show various information about the system or the program during the execution which might helpful for developers.  The output of `info` subcommand will look like following:

    $ uftrace info
    # system information
    # ==================
    # program version     : uftrace v0.6
    # recorded on         : Tue Sep 20 17:10:18 2016
    # cmdline             : uftrace record foobar 
    # cpu info            : Intel(R) Core(TM) i7-2640M CPU @ 2.80GHz
    # number of cpus      : 4 / 4 (online / possible)
    # memory info         : 7.8 / 15.5 GB (free / total)
    # system load         : 0.06 / 0.11 / 0.18 (1 / 5 / 15 min)
    # kernel version      : Linux 4.7.3-2-ARCH
    # hostname            : danjae
    # distro              : "Arch Linux"
    #
    # process information
    # ===================
    # number of tasks     : 2
    # task list           : 22073, 22071
    # exe image           : /home/namhyung/tmp/foobar
    # build id            : 889a0105318f2c41aa5e9380fb480d2cc4f362b4
    # exit status         : exited with code: 0
    # cpu time            : 0.003 / 0.000 sec (sys / user)
    # context switch      : 2 / 1 (voluntary / involuntary)
    # max rss             : 3172 KB
    # page fault          : 0 / 198 (major / minor)
    # disk iops           : 0 / 16 (read / write)
