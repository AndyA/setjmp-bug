According to http://pubs.opengroup.org/onlinepubs/7908799/xsh/setjmp.html:

> All accessible objects have values as of the time longjmp() was called,
> except that the values of objects of automatic storage duration which
> are local to the function containing the invocation of the corresponding
> setjmp() which do not have volatile-qualified type and which are changed
> between the setjmp() invocation and longjmp() call are indeterminate.

However both gcc and clang seem to have difficulty with volatile
auto pointers.

With any optimisation selected (tried -O1, -O2, -O3) both gcc and clang
optimise away the call to malloc in the following function:


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

When compiled with -g and no optimisation both gcc and clang print the
value of p as expected.

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
the c argument, gcc generates the call to malloc() and p has the
expected value. Calling it from main() like this 

```c
int main(int argc, char *argv[]) {
  test_setjmp(argc < 1000);
  return 0;
}
```

results in the expected behaviour even when compiled with -O3.

However clang always seems to lose the value of p regardless of how 
non-deterministic the call to longjmp() is.

You can reproduce the problem by typing

    $ make

in this directory. Here's what I get:

    $ make
    bash test.sh
    Testing with: gcc (Debian 4.7.2-5) 4.7.2
      Testing sjb-gcc-g
        p=0x118e010 (longjmp always)
        p=0x118e080 (longjmp depends on constant true arg)
        p=0x118e150 (longjmp depends on argc < 1000)
      Testing sjb-gcc-O1
        p=(nil) (longjmp always)
        p=0xf5d010 (longjmp depends on constant true arg)
        p=0xf5d0e0 (longjmp depends on argc < 1000)
      Testing sjb-gcc-O2
        p=(nil) (longjmp always)
        p=(nil) (longjmp depends on constant true arg)
        p=0x685010 (longjmp depends on argc < 1000)
      Testing sjb-gcc-O3
        p=(nil) (longjmp always)
        p=(nil) (longjmp depends on constant true arg)
        p=0x16e4010 (longjmp depends on argc < 1000)
    Testing with: Debian clang version 3.0-6.1 (tags/RELEASE_30/final) (based on LLVM 3.0)
      Testing sjb-clang-g
        p=0x70b010 (longjmp always)
        p=0x70b080 (longjmp depends on constant true arg)
        p=0x70b150 (longjmp depends on argc < 1000)
      Testing sjb-clang-O1
        p=0x3e8 (longjmp always)
        p=0x7d0 (longjmp depends on constant true arg)
        p=0xbb8 (longjmp depends on argc < 1000)
      Testing sjb-clang-O2
        p=0x3e8 (longjmp always)
        p=0x7d0 (longjmp depends on constant true arg)
        p=0xbb8 (longjmp depends on argc < 1000)
      Testing sjb-clang-O3
        p=0x3e8 (longjmp always)
        p=0x7d0 (longjmp depends on constant true arg)
        p=0xbb8 (longjmp depends on argc < 1000)

Is this a bug?

andy@hexten.net

