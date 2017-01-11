# Display function arguments
The uftrace supports function arguments and return value to be recorded and displayed (during replay).  Note that binaries compiled with `-finstrument-functions` cannot access to arguments yet.

## Basic usage
Let's start with the following simple example.

```
$ cat add1.c
#include <stdio.h>
#include <stdlib.h>

int add1(int n)
{
  return n + 1;
}

int main(int argc, char *argv[])
{
  int n = 1;

  if (argc > 1)
    n = atoi(argv[1]);

  add1(n);
  add1(n + 1);

  return 0;
}

$ gcc -pg add1.c
```

To see (first) argument of the add1 function, you can use `-A` option.  For now, uftrace doesn't know how to save and display arguments so user should give the information.  For simple cases, you can use "argN" (where N is a position of argument) specifier to select argument.  For instance:

```
$ uftrace -A add1@arg1  a.out
# DURATION    TID     FUNCTION
   1.737 us [13231] | __monstartup();
   1.036 us [13231] | __cxa_atexit();
            [13231] | main() {
   1.237 us [13231] |   add1(1);
   0.207 us [13231] |   add1(2);
   3.033 us [13231] | } /* main */
```

It has a similar syntax of function trigger.  A function name followed by "@" and then argument specifier.  The `arg1` means the first (and the only) argument of the given function ("add1").  The arguments are treated as an integral number (or a pointer) by default.  Argument specifier can have optional type modifier (after a "/") to change this behavior.  This is similar to "printf(3)" and will be used to print the value when replay.  For example, you can record and show them as an hexadecimal number using "x" modifier.

```
$ uftrace -A add1@arg1/x  a.out
# DURATION    TID     FUNCTION
   1.937 us [13283] | __monstartup();
   1.117 us [13283] | __cxa_atexit();
            [13283] | main() {
   1.297 us [13283] |   add1(0x1);
   0.200 us [13283] |   add1(0x2);
   3.113 us [13283] | } /* main */
```

Note that the arguments are now have '0x' prefix.  Actually uftrace tries to do its best to display arguments by default.  If no type modifier is given, small integer values (including negatives) will be displayed as decimal and big integers will be in hexadecimal.  Let's pass -1 as an argument instead of 1:

```
$ uftrace -A add1@arg1  a.out -1
# DURATION    TID     FUNCTION
   1.853 us [13496] | __monstartup();
   1.154 us [13496] | __cxa_atexit();
            [13496] | main() {
   1.370 us [13496] |   atoi();
   1.120 us [13496] |   add1(-1);
   0.216 us [13496] |   add1(0);
   4.809 us [13496] | } /* main */
```

The atoi function was called to convert the string to integer.  So the first argument of the atoi is a pointer and will have a big number.  You can use the -A option more than once:

```
$ uftrace -A add1@arg1 -A atoi@arg1  a.out -1
# DURATION    TID     FUNCTION
   1.705 us [13522] | __monstartup();
   1.180 us [13522] | __cxa_atexit();
            [13522] | main() {
   2.303 us [13522] |   atoi(0x7fff3b840d74);
   0.299 us [13522] |   add1(-1);
   0.174 us [13522] |   add1(0);
   4.636 us [13522] | } /* main */
```

As you can see, it's printed as a hex number (without the 'x' modifier).  But it's actually a pointer to string so you can use 's' modifier to show it as a string instead:

```
$ uftrace -A atoi@arg1/s  a.out -1
# DURATION    TID     FUNCTION
   1.870 us [13572] | __monstartup();
   1.124 us [13572] | __cxa_atexit();
            [13572] | main() {
   2.636 us [13572] |   atoi("-1");
   0.404 us [13572] |   add1();
   0.184 us [13572] |   add1();
   5.347 us [13572] | } /* main */
```

Beware when using the 's' modifier since it can crash your program if the given address is not valid.

Also you can specify size (in bit) of arguments.  For example, "i32" means a 32-bit signed integer (int) and "f64" means a 64-bit floating-point number (double).  The string and character type modifiers don't accept the size.  This can be used to the return value as well.  With `-R` option uftrace displays the return value:

```
$ uftrace -A atoi@arg1/s -R atoi@retval/x32  a.out -1
# DURATION    TID     FUNCTION
   1.877 us [13672] | __monstartup();
   1.183 us [13672] | __cxa_atexit();
            [13672] | main() {
   2.436 us [13672] |   atoi("-1") = 0xffffffff;
   0.186 us [13672] |   add1();
   0.140 us [13672] |   add1();
   5.330 us [13672] | } /* main */
```