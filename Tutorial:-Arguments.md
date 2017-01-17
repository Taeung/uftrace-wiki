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

Also you can specify size (in bit) of arguments.  For example, "i32" means a 32-bit signed integer (int) and "f64" means a 64-bit floating-point number (double).  The string and character type modifiers ('s' and 'c' respectively) don't accept the size.  This can be used to the return value as well.  With `-R` option uftrace displays the return value:

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

Note that it needs "retval" specifier for return value rathen than "argN".

It's also possible to give more than one argument specifier at once.

```
$ uftrace -A main@arg1,arg2  a.out
# DURATION    TID     FUNCTION
   2.070 us [14268] | __monstartup();
   1.154 us [14268] | __cxa_atexit();
            [14268] | main(1, 0x7ffee7efc888) {
   0.186 us [14268] |   add1();
   0.117 us [14268] |   add1();
   3.190 us [14268] | } /* main */
```

## Pattern matching and library calls
Instead of giving full function name, you can use a regex pattern to match multiple functions.  When uftrace detects any letter used in regex special characters it treat the name as a regex pattern.  Following example will show arguments of functions which their name start with 'a':

```
$ uftrace -A ^a@arg1  a.out
# DURATION    TID     FUNCTION
   1.973 us [ 8833] | __monstartup();
   1.140 us [ 8833] | __cxa_atexit();
            [ 8833] | main() {
   1.040 us [ 8833] |   add1(1);
   0.210 us [ 8833] |   add1(2);
   2.766 us [ 8833] | } /* main */
```

One way to set all functions to display the argument is to use '.' as a pattern.  As regex matches it to any character and it ok to match partially, it would show argument of all(?) functions.

```
$ uftrace -A .@arg1  a.out
# DURATION    TID     FUNCTION
   1.806 us [ 9467] | __monstartup();
   1.117 us [ 9467] | __cxa_atexit();
            [ 9467] | main(1) {
   0.237 us [ 9467] |   add1(1);
   0.170 us [ 9467] |   add1(2);
   2.913 us [ 9467] | } /* main */
```

As you can see, main and add1 functions both showed the first argument.  But what about "__monstartup" and "__cxa_atexit"?

The answer is that they are functions in a other library (module) not in the main executable (the a.out binary).  In fact, those library function are called through PLT (Procedure Linkage Table) and uftrace intercepts the calls at PLT.  When pattern matching is done, it first tries to match the pattern to functions in the main binary.  If it found no match, it then looks up the PLT function calls.  This was changed in the latest code (in git commit 06309be) to match PLT functions unless a specific module name was given like below.

If you want to specify functions only in the main binary or PLT, you can give the module name after the "@" sign.  The module name is a (base)name of the file or "PLT".  Below shows arguments of functions only in the a.out binary:

```
$ uftrace -A .@a.out,arg1  a.out 3
# DURATION    TID     FUNCTION
   3.086 us [22462] | __monstartup();
   1.263 us [22462] | __cxa_atexit();
            [22462] | main(2) {
   1.173 us [22462] |   atoi();
   0.180 us [22462] |   add1(3);
   0.116 us [22462] |   add1(4);
   3.446 us [22462] | } /* main */
```

Similarly, below shows arguments of the PLT functions only:

```
$ uftrace -A .@PLT,arg1  a.out 3
# DURATION    TID     FUNCTION
   2.027 us [22484] | __monstartup(0x4004e0);
   1.387 us [22484] | __cxa_atexit(0x7f03e34aa5b0);
            [22484] | main() {
   1.275 us [22484] |   atoi(0x7fffcd203d7e);
   0.158 us [22484] |   add1();
   0.173 us [22484] |   add1();
   3.692 us [22484] | } /* main */
```

## Floating-point arguments and more
Some CPU architectures, notably x86_64, pass floating-pointer arguments differently and the above way of specifying arguments won't work for them.  Thus uftrace provides "fpargN" syntax for the floating-point numbers.

Note that the index is counted separately to the normal arguments.  Let's look at the example below:

```
$ cat circle.c
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

double circumference(int radius, double pi)
{
  return 2 * pi * radius;
}

double area(int radius, double pi)
{
  return pi * radius * radius;
}

int main(int argc, char *argv[])
{
  int r = 1;

  if (argc > 1)
    r = atoi(argv[1]);

  circumference(r, M_PI);
  area(r, M_PI);
  return 0;
}

$ gcc -o circle -pg circle.c -lm
```

The circumference() and area() of the circle passes an integer argument and a floating-point argument.  You can see them using following command.

```
$ uftrace -A 'circumfence|area@arg1,fparg1' -R '^[ac]@retval/f64' circle
# DURATION    TID     FUNCTION
   2.059 us [28090] | __cxa_atexit();
            [28090] | main() {
   1.384 us [28090] |   circumference(1, 3.141593) = 6.283185;
   0.348 us [28090] |   area(1, 3.141593) = 3.141593;
   3.552 us [28090] | } /* main */
```

As you can see the second argument (pi) is the first floating-point argument, hence the "fparg1".  The return value specification is similar but it doesn't have a separate "fpretval" and uses the "f" modifier instead.