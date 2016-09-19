#uftrace usage
The uftrace tool consists of several subcommands.  We'll see how to use them with simple examples in this document.

## Getting started
The first subcommand to look at is the `live`.  It's a default subcommand and will be used if you don't give other subcommand when running uftrace.  It's basically same as running `record` and then `replay` subcommands in a row.  That means it'd show the output after a program (given on the command line) finished.  Below is the familiar "hello world" program.

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

Note that it showed the puts which is a library function you didn't write as well as your own functions in the program.  Of course it doesn't show the internals of the "puts" but it's good to see how long does the "puts" take.  It uses a technique called "PLT hooking" which redirects functions called from a dynamically-linked program.  While it's very powerful (so it's enabled by default), it comes with its overhead too.  So if you don't want to see them and/or don't want to take the overhead, you can use `--no-libcall` option for `uftrace record` to disable it.  As `live` subcommand can take all options the `record` takes, you can run like this:

    $ uftrace --no-libcall hello
    Hello world
    # DURATION    TID     FUNCTION
       7.534 us [26714] | main();

Now you can see only the "main" function - you wrote it only, right? :)   The "Hello world" in the output shows that it actually calls the "puts" function but it didn't show up in the output from uftrace.  Also, you might notice that the "main" function is shown on a single line now.  This is because uftrace shows leaf functions which don't call other functions (from the uftrace's perspective) in the compact format.
