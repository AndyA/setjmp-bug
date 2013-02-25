According to http://pubs.opengroup.org/onlinepubs/7908799/xsh/setjmp.html:

> All accessible objects have values as of the time longjmp() was called,
> except that the values of objects of automatic storage duration which
> are local to the function containing the invocation of the corresponding
> setjmp() which do not have volatile-qualified type and which are changed
> between the setjmp() invocation and longjmp() call are indeterminate.

However both gcc and clang seem to have difficulty with volatile
auto pointers.

With any optimisation selected both gcc and clang optimise away the call
to malloc in the following function:

```c
static void test_setjmp(void) {
  volatile void *p;
  if (!setjmp(jb)) {
    p = malloc(100);
    longjmp(jb, 1000);
  }
  printf("p=%p\n", p);
}
```

clang then reports the value of p as 1000 (0x3e8) while gcc reports
it as NULL.

If we make the longjmp() call conditional on an argument that's passed
in like this:

```c
static void test_setjmp(int c) {
  volatile void *p;
  if (!setjmp(jb)) {
    p = malloc(100);
    if (c) longjmp(jb, 1000);
  }
  printf("p=%p\n", p);
}
```

Then, depending on whether the compiler can figure out the value of
the c argument gcc generates the call to malloc() and p has the
expected value. Calling it from main() like this 

```c
int main(int argc, char *argv[]) {
  test_setjmp(argc < 1000);
  return 0;
}
```

results in the expected behaviour even when compiled with -O3.

However clang always seems to lose the value of p regardless of how non-
deterministic the call to longjmp() is.

You can reproduce the problem by typing

    $ make

in this directory.

Is this a bug?

andy@hexten.net

